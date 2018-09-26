
# ansible-role-docker-tls

## Securing Access to Docker with TLS

This role provides a way to protect access to a Docker host over a
network.

It supports:

* Self-signed certificates
* Accessing multiple hosts from `controller`
* Preventing bi-directional access
* Remotes can access their own TCP socket

## How Docker works with TLS

It is *very* important not to leave a Docker TCP port exposed over a
network, such as the internet, without somehow limiting the clients
that can connect.

Docker containers are all launched in `root` context and it is very
easy for bad-guys to get root access to the Docker daemon with an open
TCP socket that is not protected with TLS.

## Required Components

* Certificate authority private key and certificate
* Client private key and certificate
* Server private key and certificate

The client and server certificates are both signed with the
certificate authority key we create in this role.

## Accessing Many Daemons From One `Controller`

I needed to be able to secure several Docker daemons in remote
locations, and most of the example code I found online did not support
this.

All the Ansible code I could find worked with multiple hosts, of
course, but each host gets it's own CA certificate, and can access
it's own daemon, but not others.

Note that I am definitely *not* a TLS expert by any means, and it took
me a while to understand I needed to copy the CA certificate and key
created on the localhost to all of the hosts in the group, so that
client and server certificates could all be signed by the same private
key.

I don't really understand, but creating a private key and CA
certificate pair with the same code on two hosts seconds apart results
in different keys.

So the sequence is:

1. Create CA private key, CSR and certificate on the localhost.

2. Copy the cert and key pair created on the localhost to all other
hosts.

3. Create client keys and certs, and server keys and certs on all
   other hosts, signing them with the CA key created on the localhost
   and copied over.

4. Install CA certificate where the Docker client and daemon need it,
on all hosts.

5. Copy client keys and certs to /home/<user>/.docker on all hosts.

6. Copy server keys and certs to where the Docker daemon needs them on
all hosts, and include the CA certificate in this step.

After the above steps you will have:

```
/home/<user>/.docker/
ca.pem
cert.pem
key.pem

/etc/docker/ssl/
ca.pem
server-cert.pem
server-key.pem
```

Note that where the servercomponents are installed to is controlable.


## Variables

These are the variables in `defaults/main.yml`:


tmp_path: /tmp

Where the role creates files before they are installed/copied.

client_certificate_path: "/home/{{ ansible_user }}/.docker"

Where client components are installed.

server_certificate_path: /etc/docker/ssl

Where Docker daemon required components are installed.

subjectAltName: "IP:127.0.0.1"

The above is a critical variable and controls which hosts can access
the daemon whose server key and certificate were creatd with this
entry. See below.

# -subj fields
country: GB
state_or_province: London
locality: London
organization: Global
organizational_unit_name: IT

The subj fileds are entries in the client and server certificates.

skip_ca_cert_and_key_copy: yes

If the above is set to 'yes', then the copy from the localhost to a
remote host is kipped. This has the effect of signing that hosts
client and server certificates with a different private key, as
discussed above. So it will not be possible to access that host from the
controller.

skip_docker_service_override: no

Setting the above to 'true' will skip writing of a `dockerd` `systemd`
overrride file.

The override file is created from a template and has all the `--tls`
flags set to properly start the Docker daemon for TLS.

skip_docker_restart: no

Setting the above to 'true' will skip reloading the `systemd daemon`
and restarting the Docker daemon.

## subjectAltName

As mentioned above, this is a very critical variable in the process of
generating our server certificate.

It can contain both DNS and IP address entries.

By default, with this variable set to:

IP:127.0.0.1

Then only the localhost can access the socket.

If one of your hosts is called `docker001`, then setting this variable
to:

DNS:docker001,IP:127.0.0.1

Makes it possible to connect from the named host and the localhost
address.

I don't think there is a limit to the number of entries.

I don't know of any wildcard functionality here, but you would not
want that for a Docker daemon anyway.

## Host Level Variables


