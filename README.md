# README #

Consortia Hyperledger Fabric Ansible playbooks

### What is this repository for? ###

Holds ansible playbooks for setting up hyperledger fabric network. The playbooks are imported and used in Asnible Tower projects.
The repository provides a different branch for every Hyperledger Fabric release.


### How do I get set up? ###


### Prerequisites ###

Linux machine - version to do - tested with ubuntu-noble-24.04-amd64-server-20240801

SSH access to the machine
Software installed on the machine 
- docker - what version ?
- docker - https://docs.docker.com/engine/install/ubuntu/
- ACL Must install ACL sudo apt-get install acl
Initialize docker swarm: docker swarm init
Add fabric user to the docker user group
Domain name for the Fabric Ops Console
Domain name for the HLF nodes 




Load Balancer with TLS to be used for the Fabric Ops Console

Installed ansible locally (or AWX)






### Steps ###

#### Configure
central config file with all the config, point to the file withinline comments
if running locally with ansible-playbooks command
- install ansible: https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

#### Create wokring Linux user

Playbook: create-users.yml
Command: ansible-playbook create-users.yml  -e "target_machine=node" --extra-vars "@vars/config.yml"
Description: Creates a dedicated fabric user and user group to setup, install and run the steps

#### Linux Prerequisites installations and setup
- Docker
    - Install, 
        - post install docker group 
          sudo usermod -aG docker $USER
          sudo usermod -aG docker fabric
    - init 
        	docker swarm init
- Libraries
    - ACL 
        sudo apt install acl


#### Setup wokring HLF Folders

Playbook: setup-fabric-tools-folders.yml
Command: ansible-playbook setup-fabric-tools-folders.yml  -e "target_machine=node" --extra-vars "@vars/config.yml"
Description: Creates dedicated folders to install required libraries, configure the network and store the related config files and crypto materials


#### Install HLF Tools

Playbook: install-fabric-tools.yml
Command: ansible-playbook install-fabric-tools.yml  -e "target_machine=node" --extra-vars "@vars/config.yml"
Description: Installs HLF tools from HLF repository

#### Deploy Fabric Ops Console

Playbook: deploy-fabric-operations-console.yml
Command: ansible-playbook deploy-fabric-operations-console.yml  -e "target_machine=node" --extra-vars "@vars/config.yml"
Description: Configures and deploys fabric ops console
The console will use the provided TLS certs. If not TLS certs are provided the playbook will generate a self signed certs.


#### Nginx for Fabric Ops Console 

This step can be skipped if a dedicated load balancer is already availabe and configured to handle the networking for the fabric operations console.
Playbook: deploy-nginx-fabric-operations-console.yml
Command: ansible-playbook deploy-nginx-fabric-operations-console.yml  -e "target_machine=node" --extra-vars "@vars/config.yml"
Description: Configures and deploys an nginx container to provide TLS connection for fabric operations console

#### Deploy MSP CA

Playbook: deploy-msp-ca.yml
Command: ansible-playbook deploy-msp-ca.yml  -e "target_machine=node" --extra-vars "@vars/config.yml"

You need to open the ports (API and operations ports) of the Fabric CA so it could be reached by the Fabric Operations Console.

The Fabirc CA provides WEB APIs, it could be exposed to the outside world using a load balancer.


#### Register and enroll the MSP Peer with CA

Playbook: register-enroll-msp-peer.yml
Command: ansible-playbook register-enroll-msp-peer.yml  -e "target_machine=ca" --extra-vars "@vars/config.yml"


#### Deploy MSP Peer

Playbook: deploy-msp-peer.yml
Command: ansible-playbook deploy-msp-peer.yml  -e "target_machine=ca" --extra-vars "@vars/config.yml"

#### Open the HLF peer to internet

Expose the peer port to internet

#### Export MSP, CA, Peer config and users

#### Import MSP, CA, Peer, users to Fabric Ops Console




