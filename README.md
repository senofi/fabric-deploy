# README #

Deploy Hyperledger Fabric Ansible playbooks

### What is this repository for? ###

The repository contains ansible playbooks that automate the deployment of Fabric Operations Console, Hyperledger CA and HLF peer nodes on a prepared, configured and provisioned infrastructure on Linux machines in Docker containers.

The networking and infrastructure resource provisioning is environment/cloud specific and should be designed and implemented upfront.

### Deployment Overview

The HLF deployment can be done on various types of infrastructures (cloud or on premise). Usually a typical deployment involves running the HLF components (fabric ca, peer, orderer) in containers like docker, k8s pods or any other type of cloud specific containerization capability. Depending on the used infrastructure the deployment model may get quite complex as its topology usually depend on multiple policies (e.g. security related).
One of the challenges within a company's IT department is exposing the API port of their HLF peer to internet. 

The diagram below is a simplified view of a typical HLF deployment using basic networking and docker containers.

![](docs/LandscapeModel.jpg?raw=true)

The purpose of the load balancer(s) is to route the network traffic to the HLF containers. The communication on the HLF ports is TLS enabled so there is no need to terminate the TLS connection at the load balancer.
It is a good practice to use a dedicated subdomain to define the HLF endpoints (peer, ca, fabric operation console) on the TLS port 443. There is no need to provision your own TLS certs as those are generated during the deployment step.

It is also possible to route the inbound traffic directly to the machine (without using load balancers) where the HLF containers are deployed. However that requires a public IP(s) and proper management of multiple ports. In some cases that may be a security concern.

The infrastructure should be prepared, configured and provisioned upfront before deploying the HLF nodes. That includes the preparation/configuration of the DNS names of the HLF endpoints, provision of the machines, provision of the load balancers and their respective configurations for inbound traffic rules, network shares, etc.

### Prerequisites ###
1. Linux instance(s) where the HLF containers will be deployed. If more than one machines are used, repeat the step on each of the provisioned machines.
    - SSH access to those instances is required from the environment where the ansible playbooks will be run (local machine or AWX)
    - Create a dedicated Linux group and user (default user/group name "fabric") for use by the ansible playbooks. You may do it manually or run the playbook create-users.yml.
    ```
    sudo useradd fabric
    ```
     
    - Software packages:
        - Docker - https://docs.docker.com/engine/install/ubuntu/ , make sure to follow the post install steps: https://docs.docker.com/engine/install/linux-postinstall/
        - ACL library - e.g. 
        ```
        sudo apt-get install acl"
        ```
    - Configure docker group to add the newly created "fabric" user  -e.g. 
    ```
    sudo usermod -aG docker fabric
    ```
    - Initialize docker swarm on the provisioned instances by running the command: 
    ```
    sudo docker swarm init
    ```
    - A network file folder may be created and attached to the machine. The folder can serve as a common shared location of the fabric configurations and crypto material. This is useful when multiple machines are used to deploy additional fabric peers.

2. DNS names for HLF endpoints and Load Balancer(s) to handle the inbound traffic to the HLF nodes
    - Define domain names for the HLF nodes: Fabric operation console, fabric peer, fabric peer webproxy, Fabric CA, Fabric TLS CA
    - Provision Load Balancers for the HLF endpoints, configure the inbound traffic rules (no TLS termination). You may request for a public trusted CA and use those TLS certs for the Fabric operations console
    - Configure the DNS records with your trusted DNS provider

3. Local machine
    - Software packages:
        - ansible (ansible-playbook): https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#pipx-install
    - Configure ansible - e.g. create ssh config entry for the provisioned instances, configure ansible hosts file with your inventory/instances (https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)
You may also use AWX (https://ansible.readthedocs.io/projects/awx/en/latest/) to execute the ansible-playbooks instead of running them from your local machine with ansible.


### Steps ###

#### Prepare the ansible configuration var file

You may use vars/config.yml as a template. You may go through the file comments for better understanding for each var config.

The target_machine variable value must be configured and should be the name of the remote machine as defined in the ansible hosts file.

Execute the following ansible playbooks to deploy the HLF components

### Ansible Playbook ###
#### Setup working HLF Folders

Playbook: setup-fabric-tools-folders.yml
Command: 
```
ansible-playbook setup-fabric-tools-folders.yml --extra-vars "@vars/config.yml"
```
Note: 
Creates dedicated folders to install required libraries, configure the network and store the related config files and crypto materials.
The path to the root working folder is configured in the supplied config.yml. It can point to a network file share if multiple machines are used.
If using multiple machines, run this playbook on each one of them. You may change the remote machine by adjusting the target_machine variable.


#### Install HLF Tools

Playbook: install-fabric-tools.yml
Command: 
```
ansible-playbook install-fabric-tools.yml --extra-vars "@vars/config.yml"
```

Note: 
Installs HLF tools from HLF repository.
If using multiple machines, run this playbook on each one of them. You may change the remote machine by adjusting the target_machine variable.


#### Deploy Fabric Ops Console

Playbook: deploy-fabric-operations-console.yml
Command: 
```
ansible-playbook deploy-fabric-operations-console.yml --extra-vars "@vars/config.yml"
```
Note: 
Configures and deploys fabric ops console. The console will use the configured TLS certs. If no TLS certs are provided the playbook will generate and use a self signed TLS certs.

#### Deploy MSP CA and TLS CA

Playbook: deploy-msp-ca.yml
Command: 
```
ansible-playbook deploy-msp-ca.yml --extra-vars "@vars/config.yml"
```
Note:
This playbook will deploy two CA servers to issue the certificates and TLS certificates of the HLF MSP. Each MSP has its own set of CA servers.
Run this playbook on the machine where the CA should be deployed. You may change the remote machine by adjusting the target_machine variable.

#### Register and enroll the MSP Peer with Fabric CA and Fabric TLS CA

Playbook: register-enroll-msp-peer.yml
Command: 
```
ansible-playbook register-enroll-msp-peer.yml  --extra-vars "@vars/config.yml"
```
Note:
This playbook will register and enroll the MSP peer identity with the MSP CA servers. 
Run this playbook on the machine where the peer will be running. If the peer machine is different from the CA machine you should make sure to share the certs required to connect to the CA servers.
You may change the remote machine by adjusting the target_machine variable.


#### Deploy MSP Peer

Playbook: deploy-msp-peer.yml
Command: 
```
ansible-playbook deploy-msp-peer.yml --extra-vars "@vars/config.yml"
```
This playbook will deploy a HLF peer in docker container. Run this playbook on the machine where the peer will be running. 
You may change the remote machine by adjusting the target_machine variable.

#### Export MSP, CA, Peer config and users

Run the below playbooks on the machine where the CA servers are running.

Playbook: export-console-msp-admin-users.yml
Command: 
```
ansible-playbook export-console-msp-admin-users.yml --extra-vars "@vars/config.yml"
```
Note: 
Generates json files on the remote machine that contains the crypto of the admin users of the Fabric peer and Fabric CA / TLS CA servers..
The files can be downloaded locally and imported manually inside the fabric operations console wallet.
The generated json file are:
- Peer admin: {{ data_folder }}/fabric-console/export/users/{{ msp_name }}/fabric_user_{{ msp_name }}_{{ msp_peer_admin_user }}.json
- CA admin: {{ data_folder }}/fabric-console/export/users/{{ msp_name }}/fabric_user_{{ msp_name }}_{{ ca_name }}_{{ msp_ca_admin_user }}.json
- TLS CA admin: {{ data_folder }}/fabric-console/export/users/{{ msp_name }}/fabric_user_{{ msp_name }}_{{ caTls_name }}_{{ msp_tls_ca_admin_user }}.json

The paths to the generated files on the remote machine can be found inside the playbook log.

Playbook: export-console-msp-config.yml
Command: 
```
ansible-playbook export-console-msp-config.yml --extra-vars "@vars/config.yml"
```
Note: 
Generates json files on the remote machine that contains the definition and connection details of the Fabric CA, Fabric TLS CA and MSP.
The files can be downloaded locally and imported manually inside the fabric operations console (import organization and import CA nodes).

The generated json file are:
- MSP definition: {{ data_folder }}/fabric-console/export/msp/fabric_console_msp_{{ msp_name }}.json
- CA definition: {{ data_folder }}/fabric-console/export/msp/fabric_console_msp_{{ msp_name }}.json
- TLS CA definition: {{ data_folder }}/fabric-console/export/msp/fabric_console_msp_{{ msp_name }}.json
The paths to the generated files on the remote machine can be found inside the playbook log.


Playbook: export-console-peer-config.yml
Command: 
```
ansible-playbook export-console-peer-config.yml --extra-vars "@vars/config.yml"
```
Note:
Generates a json file on the remote machine that contains the definition and connection details of the Fabric peer.
The file can be downloaded locally and imported manually inside the fabric operations console (import peer node).

The generated json file is:
- Per definition: {{ data_folder }}/fabric-console/export/peer/fabric_console_peer_{{ msp_name }}_{{ peer_name }}.json

The exact path to the generated file on the remote machine can be found inside the playbook log.


#### Import MSP, CA, Peer, users to Fabric Ops Console

The generated json files from the previous step can be downloaded locally from the remote machine. Those json files can be then imported inside the fabric operation console so that the deployed Fabric assets can be operated.

The following list represents the imports in fabric operation console
- Wallet admin identities import
    - Wallet -> Add identity -> Upload JSON -> Add identity
    - Repeat the above step to import the admin identities stored in the following files
        - Peer admin json file: fabric_user_{{ msp_name }}_{{ msp_peer_admin_user }}.json
        - Certificate Authority admin json file: fabric_user_{{ msp_name }}_{{ ca_name }}_{{ msp_ca_admin_user }}.json
        - TLS Certificate Authority admin json file: fabric_user_{{ msp_name }}_{{ caTls_name }}_{{ msp_tls_ca_admin_user }}.json
- Peer Node import
    - Nodes -> Import peer -> Add file -> Add peer
    - Choose the peer config json file: fabric_console_peer_{{ msp_name }}_{{ peer_name }}.json
    - Nodes -> Peer ( select the imported peer ) -> Associate Identity
    - Choose the peer admin identity from the wallet
- CA Nodes import (CA and TLS CA)
    - Nodes -> Import Certificate Authority -> Add file -> Add Certificate Authority
    - Repeat the above step to import the CA nodes definition as stored in the following files:
        - Certificate Authority json file:  fabric_console_ca_{{ msp_name }}.json
        - TLS Certificate Authority json file: fabric_console_tlsca_{{ msp_name }}.json
    - For each imported CA node, choose the corresponding admin identity (CA admin and TLS CA admin) from the wallet
    - Nodes -> Peer ( select the imported peer ) -> Associate Identity
- MSP definition import
    - Organizations -> Import MSP Definition -> Add file -> Import MSP Definition
    - Choose the MSP definition json file: fabric_console_msp_{{ msp_name }}.json

After the import the fabric operations console should be able to reach the Fabric Nodes. In case of successful connection the console should mark the nodes with a green light. 

The console will let you operate your nodes, join HLF networks, deploy your smart contracts, enroll application identities, etc. For more details about the console visit: https://github.com/hyperledger-labs/fabric-operations-console/tree/main/docs