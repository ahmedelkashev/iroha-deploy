# Description
[This role](https://github.com/hyperledger/iroha-deploy/edit/master/ansible/roles/iroha-docker) deploys multiple replicas of Iroha containers (one Iroha peer per container) on remote hosts. Each Iroha peer can communicate with others in two ways:
  - using public IP addresses or hostnames set in inventory list OR
  - using private IP addresses of the Docker overlay network

The first one is easier to implement since it does not require preliminary configuration of the remote hosts. Just make sure that network ports are not firewalled. You can check the port list in the generated Docker Compose file (`docker-compose.yml`) after deployment.

This option is enabled by default.

The second one can be used when there exists an overlay network between the hosts. In short, overlay network allows for Docker containers to communicate using a single subnet. Such that each container would have a unique IP address in that subnet. Learn more in official Docker documentation (https://docs.docker.com/network/overlay). Overlay network can be created if your instance is part of a Swarm cluster. Another method does not involve creating a Swarm cluster but requires a distributed key-value storage. [There is a guide](https://docker-k8s-lab.readthedocs.io/en/latest/docker/docker-etcd.html) on how to create such overlay network.

This method is also suitable for local-only deployments and does not require any key-value storage.

# Requirements
  Tested on Ubuntu 16.04, 18.04
  - Local:
    - python3
    - ansible(>=2.10)
  - Remote:
    - Docker (>=17.12)
    - python3
    - PIP3 modules: docker, docker-compose

    There is a role for setting up a remote part of the dependencies named `docker`. It works for Ubuntu OS only. Check `iroha-docker` playbook.

**Note:**
> `docker.io` package from Ubuntu repos will not work. Either use Ansible role or install Docker following official instructions for your OS flavor.

# Quick Start
1. Install Ansible
    ```
    pip3 install ansible
    ```
2. Run ansible 

    ```
    cd ../../ && ansible-playbook -b -e 'ansible_ssh_user=ubuntu' -i '192.168.122.109,' playbooks/iroha-docker/main.yml`
    ```

This will deploy 4 Iroha Docker containers along with one Postgres containers on the remote host specified in `-i`. Remote user is `ubuntu`. Torii port of each container is exposed on the host. Iroha peer can be communicated over port defined in `iroha_torii_port` variable (50051 by default). Overall, each host will listen the following port range: `iroha_torii_port` ... `iroha_torii_port` + *number-of-containers* - 1.

**Note:**
> This command escalates privileges on a remote host during the run. It is required to be able to spin up Docker containers. We recommend to run the playbook using a passwordless remote sudo user.

# Initial configuration

See `defaults/main.yml` file to get more details about available configuration options.

# Examples
**Example 1**
<!-- TODO: Cover more example cases -->
Deploying Iroha to localhost with default number of replica: 
1. run ansible command:

    ```
    cd ../../ && ansible-playbook -e 'iroha_network=internal' -c local  -i 'localhost,' playbooks/iroha-docker/main.yml -b -K`
    ```
    `-K` - parameter will ask you for sudo password: 
    ```
    BECOME password: 
    ```

This will deploy 4 Iroha Docker containers along with one Postgres containers on the local host. 
Torii port of each container is exposed on the host. 
Iroha peer can be communicated over port defined in `iroha_torii_port` variable (50051 by default). 
Overall, each host will listen the following port range: `iroha_torii_port` ... `iroha_torii_port` + *number-of-containers* - 1.
Iroha peer will communicate with each over through docker network. And will use docker container name for it. 

**Example 2**

Deploying 6 Iroha peers on two remote hosts communicating using public IP addresses. With 2 and 4 replicas on each host respectively.

1. Create inventory list containing IP addresses (or hostnames if they are mutually resolve-able on both hosts) of two hosts that will run Iroha peers

    **iroha.list**
    ```
    [all]
    192.168.122.109
    192.168.122.30
    ```

    Put this file into `../../inventory/` directory.
2. Make sure you can SSH with a root account into either of these hosts using a private key.

    **Note**
    > You can also SSH with the user other than root. Make sure it can execute `sudo` without prompting for a password. Set `-u` option for `ansible-playbook` command.

3. Create two YAML files in `../playbooks/iroha-docker/host_vars` directory:

    **192.168.122.109.yml**
    ```
    iroha_replicas: 2
    iroha_service_host: True
    # this is account and private key from example genesis.block
    iroha_service_account: admin@test
    iroha_service_account_keys: [ 'f101537e319568c765b2cc89698325604991dca57b9716b58016b253506cab70' ]
    ```

    **192.168.122.30.yml**
    ```
    iroha_replicas: 4
    ```

4. Run the playbook
```
ansible-playbook -i inventory/iroha.list -b playbooks/iroha-docker/main.yml
```
**Example 3**
Deploying 3 Iroha peers on two remote hosts communicating using DNS name. With 2 and 1 replicas on each host respectively.
We will use pivate ip (`192.168.122.109, 192.168.122.30`) to connect and deploy Iroha and we will use DNS name (`iroha[1-3].example.com`) to communicate between peers.

1. Create inventory list containing IP addresses (or hostnames) of two hosts that will run Iroha peers.

    **iroha.list**
    ```
    [all]
    192.168.122.109
    192.168.122.30
    ```

    Put this file into `../../inventory/` directory.

2. Make sure you can SSH with a root account into either of these hosts using a private key.

    **Note**
    > You can also SSH with the user other than root. Make sure it can execute `sudo` without prompting for a password. Set `-u` option for `ansible-playbook` command.

3. Create two YAML files in `../playbooks/iroha-docker/host_vars` directory:
    **192.168.122.109.yml**
    ```
    iroha_replicas: 2
    iroha_service_host: True
    # this is account and private key from example genesis.block
    iroha_service_account: admin@test
    iroha_service_account_keys: [ 'f101537e319568c765b2cc89698325604991dca57b9716b58016b253506cab70' ]
    iroha_custom_hostnames: true
    iroha_hostnames: ["iroha1.example.com:10001","iroha2.example.com:10002"] # we use different port because both replica are in one server
    ```

    **192.168.122.30.yml**
    ```
    iroha_replicas: 1
    iroha_custom_hostnames: true
    iroha_hostnames: ["iroha3.example.com:10001"]
    ```

 4. Run the playbook
```
ansible-playbook -i inventory/iroha.list -b playbooks/iroha-docker/main.yml
```

**Example 4**
Deploying 6 Iroha peers on two remote hosts communicating over overlay network (Calico) using custom hostnames.

**TBD**

##### Caveats
1. If `/usr/bin/python` does not exist on a remote host, Ansible will fail with the misleading message: `... Make sure this host can be reached over ssh`. This usually happens when Ansible uses Python 3. On Ubuntu systems `/usr/bin/python3` is not symlinked to `/usr/bin/python` which Ansible expects to find. The problem can be solved by setting `ansible_python_interpreter` variable to `/usr/bin/python3`.
