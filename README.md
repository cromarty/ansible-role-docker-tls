
# ansible-role-docker-tls

## Securing Access to Docker with TLS

This role provides a way to protect access to a Docker host over a
network.

It supports:

* Self-signed certificates
* Accessing multiple hosts from single controller
* Preventing bi-directional access
* Remotes can access their own TCP socket

In the above, *controller* is the machine running your Ansible
playbook executing this role.

## How Docker works with TLS

I should point out that I am no expert on the subject of TLS.

It is *very* important not to leave a Docker TCP port exposed over a
network, such as the internet, without somehow limiting the clients
that can connect.

Docker containers are all launched in root context and it is very
easy for bad-guys to get root access to the host with an open
TCP socket that is not protected with TLS.

## Required Components

* Certificate authority private key and certificate
* Client private key and certificate
* Server private key and certificate

The client and server certificates are both signed with the
certificate authority key we create in this role.

## Accessing Many Daemons From One Controller

I needed to be able to secure several Docker daemons in remote
locations, and most of the example code I found online did not support
this.

All the Ansible code I could find worked with multiple hosts, of
course, but each host gets it's own CA certificate, and can access
it's own daemon, but not others.

As stated above I am definitely *not* a TLS expert by any means, and it took
me a while to understand I needed to copy the CA certificate and private key
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
   hosts, signing them with the CA key created on the localhost
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

```
/etc/docker/ssl/
	ca.pem
	server-cert.pem
	server-key.pem
```

Note that where the server components are installed to is controlable.

Note also it is possible to install the client components to a place
other than:

```
/home/<user>/.docker
```

By setting the environment variable `DOCKER_CERTS_PATH` on that host
to point to where these three files are stored:

* ca.pem
* cert.pem
* key.pem

Note here also that the names of these files is critical. It does not
appear to work if the CA certificate is not called `ca.pem`, the
private key is not `key.pem`, or the certificate is not `cert.pem`.

## Variables

These are the role variables:

```
privatekey_passphrase: <secret>
```

See below for an explanation of keeping the passphrase in an Ansible
vault, to keep it secret.

```
tmp_path: /tmp
```

Where the role creates files before they are installed/copied.

```
client_certificate_path: "/home/{{ ansible_user }}/.docker"
```

Where client components are installed.

```
server_certificate_path: /etc/docker/ssl
```

Where Docker daemon required components are installed.

Note that, as will be seen below, all three files needed to start the
Docker daemon in TLS mode do not need to all be in the same path, but
this role only supports a single path for all three.

When the Docker daemon is started, there are switches that will point
to the two server components and the CA certificate (see later).

```
subjectAltName: "IP:127.0.0.1"
```

The above is a *critical* variable and controls which hosts can access
the daemon whose server key and certificate were created with this
entry. See below.

```
# -subj fields
country: GB
state_or_province: London
locality: London
organization: Global
organizational_unit_name: IT
```

The subj fieleds are entries in the client and server certificates.

The values shown are the defaults provided in `defaults/main.yml`.

```
skip_docker_ca_cert_and_key_copy: no
```

If the above is set to 'yes', then the copy from the localhost to a
remote host is skipped. This has the effect of signing that hosts
client and server certificates with a different private key, as
discussed above. So it will not be possible to access that host from the
controller.

I don't see a point in this, as the whole point of TLS is to access
the Docker daemon safely over the WAN. But this will behave like all
other Ansible code I found by Googling.

```
skip_docker_service_override: no
```

Setting the above to 'true' will skip writing of a `dockerd` `systemd`
overrride file.

The override file is created from a template and has all the `--tls`
flags set to properly start the Docker daemon for TLS.

The override file is installed in:

```
/etc/systemd/system/docker.service.d/override.conf
```

Note this role *does not* support non-systemd systems.

```
skip_docker_restart: no
```

Setting the above to 'true' will skip reloading the systemd daemon
and restarting the Docker daemon.

### subjectAltName

As mentioned above, this is a very critical variable in the process of
generating our server certificate.

It can contain both DNS and IP address entries.

By default, with this variable set to:

```
IP:127.0.0.1
```

Only the localhost can access the socket.

By setting this variable as above on the localhost, bi-directional
access to it's socket from the other hosts is prevented. This is
probably what you want.

If one of your hosts is called `docker001`, then setting this variable
to:

```
DNS:docker001,IP:127.0.0.1
```

Makes it possible to connect from the named host and the localhost
IP address.

So as well as connecting to the TLS protected socket from the
controller, connecting from the remote host to it's own localhost TCP
port is possible.

I don't think there is a limit to the number of entries.

I don't know of any wildcard functionality here, but you would not
want that for a Docker daemon anyway.

I guess there *must* be wildcard functionality in TLS, otherwise it
would not be possible for Web servers to accept connection from
everybody who has the necessary certificate in their browser. Or am I
misunderstanding something?

```
docker_port: 2376
```

The default value for a Docker daemon TLS protected port is 2376.

If you have multiple remote hosts at the same IP address, and
accessible thanks to port-forwarding at that address, then this port
can be set differently for each remote host in it's corresponding:

```
host_vars/host.yml
```

## Docker daemon TLS Switches

Here is how to start the Docker daemon to support TLS.

You need to provide some switches when starting the Docker daemon, in
addition to the TCP address and port:

```
--tlsverify
```

Instructs dockerd to start needing TLS verification from any client.

And these three switches point to the other three necessary files:

```
--tlscacert=/path/to/ca.pem
--tlscert=/path/to//server-cert.pem
--tlskey=/path/to/server-key.pem
```

This is the line in the `override.conf.j2` template:

```
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:{{ docker_port }} --tlsverify --tlscacert={{ server_certificate_path }}/ca.pem --tlscert={{ server_certificate_path }}/server-cert.pem --tlskey={{ server_certificate_path }}/server-key.pem
```

You can see from the above how the variables in `defaults/main.yml`
need to be set to where these three components are installed.

Of course they can be overridden at group or host level in your playbooks.

The switch:

```
-H fd://
```

Points to the host's Unix socket.

The:

```
-H tcp://{host>:<port>
```

Switch sets where the host Docker daemon is listening.

## Keeping `privatekey_passphrase` a Secret

It is *strongly* recommended that the private key passphrase, used in
creating the CA private key and certificate are not stored in plain
text.

To do this we use an Ansible vault.

In:

```
group_vars/all
```

Create the two files:

* vars.yml
* vault.yml

The contents of `vars.yml` should look something like this:

```
---

ansible_user: "{{ vault_ansible_user }}"
ansible_become_pass: "{{ vault_ansible_become_pass }}"
privatekey_passphrase: "{{ vault_privatekey_passphrase }}"
```

The contents of `vault.yml` will then be:

```
---

vault_ansible_user: <ssh user name>
vault_ansible_become_pass: <ssh password>
vault_privatekey_passphrase: <private key passphrase>
```

Obviously replace the values in `<...>` above.

### Encrypt the Vault

Now you need to encrypt the Ansible vault like this:

```
ansible-vault encrypt vault.yml
```

You will be asked for the password twice, the second time for verification.

## Running the role in a playbook.

To run a playbook which uses the role and the Ansible vault:

```
ansible-playbook --ask-vault-pass <playbook.yml>
```

You will need for all hosts in your inventory to have the same
`ansible_user` and `ansible_become_pass` value.

I don't think it is possible to get many entries from many vaults in a
single run of a playbook, but I may be wrong.

## Notes

### Man-in-the-middle (MITM) Attacks

I read somewhere that this approach does not provide server identity
verification. So you will not be able to guarantee that the Docker
daemon you are connecting to on a remote host is the machine you think
it is.

But if there is a MITM (man-in-middle), he will just see typical
Docker requests to:

* Pull images
*  Start and stop containers
* List pulled images
* List running and stopped containers
* Inject processes into running containers

And more.

You're not going to send credit card data over the Docker daemon HTTP port.

But I am no security expert.

## How to Configure Docker Contained Services Remotely

I recently had a very difficult time finding out how to configure a
service running in Docker containers running on remote hosts with
Ansible.

### Problem

I needed to configure a MongoDB replica set with four nodes running
inside four separate Docker containers, each running on a different
host.

These four nodes are:

* Master node
* Replica 1
* Replica 2
* Arbiter

We don't need to write here anything about MongoDB terminology or
configuration, other than that I needed to start each container with
the `/sbin/init` process running as `pid 1` and then run some Ansible
code to configure MongoDB functionality.

How to address stuff inside a container running on a remote host?

Just writing:

```
mongo1 ansible_connection=docker
```

Did not seem to be enough.

The secret lies in the Ansible parameter `delegate_to:`.

Using Ansible code like this:


```
---

- hosts: mongo
  gather_facts: yes
  become: yes
  roles:
    - mymongodb
      delegate_to: "{{ inventory_hostname }}"
```

But this is not enough. In the `host_vars/``` file for the host which
represents the container, you need to put an entry like this:

```
ansible_docker_extra_args: "-H=tcp://{{ mongodb_host }}:2376 --tlsverify"
```

For example in `host_vars/mongo1.yml`. Here `mongo1` is one of the
containers containing one of my MongoDB nodes, actually the Master
node.

In the above the `mongodb_host` variable contaimns the IP address of
the Ansible host hosting the `mongo1` container.

The `--tlsverify` flag is needed when the Docker daemon on the Ansible
host hosting the container is running with TLS verification on.

Since the change which introduced the `docker_tls_verify` boolean
variable, a more complete `ansible_docker_extra_args` line might look
like this:

```
ansible_docker_extra_args: "-H=tcp://{{ mongodb_host }}:{{ docker_port }} {{ (docker_tls_verify|default(true))|ternary('--tlsverify','') }}"
```

See the contents of `templates/override.conf.j2` for `systemd` service
file override details relating to `docker_tls_verify.

Please note that if `docker_tls_verify` is set to false, then
previously created and copied certificates and keys are not removed
either from the Ansible provisioning host or from any Docker host, but
the `override.conf` file is replaced without the switches that make
TLS verification necessary, and the service is started with access
through Unix domain socket only.

I thought it was a better and more secure option to remove TCP socket
access completely when switching from `docker_tls_verify` being true
to false.


