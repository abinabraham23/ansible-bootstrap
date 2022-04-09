# Setup Ansible playground using Docker

* Create an SSH key using `ssh-keygen` to connect to docker containers (ansible hosts) using SSH.

```
ssh-keygen -b 2048 -t rsa -f id_rsa -q -N ""
```
The above command will generate a pair of key files: public key (_id_rsa.pub_) and private key (_id_rsa_).

_Note: Place both public and private key to Control Node directory. Place only public key to Managed Node directory_

* Prepare Dockerfile to build a docker image with ssh enabled and python3/other packages installed. In addition, include the newly created private and public key files and set public keys in the authorized_keys file of the root user. Refer to the Dockerfile

* Build the images for Control Node and Managed Node based on the Dockerfile in respective directories
Build under /ControlNode
```
docker build -t ansible-control .
```
Build under /ManagedNode
```
docker build -t ansible-host .
```

* Create docker network to connect/ssh from one container to another container within the network
```
docker network create -d bridge ansible
```

Note:
1. Above uses docker service discovery and we can achieve service discovery by connecting all the containers to the same network.
2. Without the docker network, ssh-ing is only possible from host to container, but not between containers.

* Run the first container as control node with specified port and hostname in ansible hosts
```
docker run --name control --hostname control.test --network ansible -p 7770:22 -d ansible-control
```

* Run the second container as host1/server1 with specified port and hostname in ansible hosts
```
docker run --name host1 --hostname host1.test --network ansible -p 7771:22 -d ansible-host
```

 * Run the third container as host2/server2 with specified port and hostname in ansible hosts
```
docker run --name host2 --hostname host2.test --network ansible -p 7772:22 -d ansible-host
```

* Test with SSH to the newly created containers from host system with private key as input and root user
```
ssh -i id_rsa -p 7770 root@localhost
ssh -i id_rsa -p 7771 root@localhost
ssh -i id_rsa -p 7772 root@localhost
```
* Test with SSH from control node container to managed node containers (as they are part of same docker network)
```
root@control:~# ssh root@host1
root@control:~# ssh root@host2
```

 __Note__: In case of any error with known-hosts, run the below command to remove the known host:
```
ssh-keygen -f "~/.ssh/known_hosts" -R "[localhost]:7771"
```

## Ansible configuration and setup on Control Node

The Ansible control node needs to be setup to communicate with the managed hosts (nodes). To do so, it requires few directives and files:

### Method 1 : Copy the repository files to ansible control node container
docker cp inventory control:/opt/ansible/
docker cp ssh.config control:/opt/ansible/
docker cp ansible.cfg control:/opt/ansible/

### Method 2 : Create the files manually on control nodes

* Create an 'inventory' file with below hostnames under group "servers"
```
[servers]
host1
host2

[servers:vars]
ansible_python_interpreter=/usr/bin/python3
```

* Create a ssh config file (ssh.config) to define the remote hosts which corresponds with running Docker containers, that resides within our local machine and it is mapped by its exposed port. (here 7771 & 7772). In addition, we disable the host key checking and with IdentityFile, specify which Private Key to use.

```
Host *
    #disable host key checking: avoid asking for the keyprint authenticity
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    #enable hashing known_host file
    HashKnownHosts yes
    #IdentityFile allows to specify exactly which private key I wish to use for authentification
    IdentityFile /root/.ssh/id_rsa

Host host1.test
    HostName localhost
    User root
    Port 7771

Host host2.test
    HostName localhost
    User root
    Port 7772
```

* Set custom inventory and ssh_connection parameter in ansible.cfg file
```
[defaults]
inventory = inventory

[ssh_connection]
transport=ssh
ssh_args= -F ssh.config
```

* Test the connectivity (Run the command from the directory that includes ansible.cfg and inventory files i.e. /opt/ansible):
```
ansible servers -m ping
```
