repositories# DEVNET-2076: Cisco Live US 2018

This repo contains the material used in the DEVNET-2076 session at Cisco Live US 2018.  It is a scaled down version of what might be used in a real environment that illustrates one possible DevOps pipeline for networks with Ansible.  

## Background

Ansible is simply a tool that automates the individual tasks that are normally manually performed by a network operations teams. Ansible has no innate intelligence for determining a good task from a bad task, so it will happily and efficiently create or destroy depending on the inventory and playbooks fed to it.  Also, IaC network automation systems are gnerally made up of components from several different sources.  Each of these sources are separately developed and versions.  A change in one of these components can perturb the entire system.  For these reasons, successful network-automation at scale should be integrated into a DevOps process.  This session outlines a prototype Devops pipeline that can be used for network automation.

## DevOps Infrastructure

Our DevOps infrastructure consists of 3 components:

* Source Control: GitHib
* Test Engine: Jenkins
* Automation Orchestrator: Ansible Tower
* Collaboration Platform: Cisco WebEx Teams

### Source Control

#### Repository Layout

The layout of the repo is pretty standard for Ansible.  The main point is in how the layout is used.  For example, Ansible allows one to specify the inventory to be used.  In this repo, we have two different inventories: test and prod. A couple of key points to remember about Ansible is that an `inventory` contains both the list of devices to automate and the key/value pairs (a.k.a Source of Truth) that defines how those devices are configured.  

```
.
├── Jenkinsfile
├── LICENSE
├── README.md
├── build-testbed.yml
├── destroy-testbed.yml
├── inventory                   |
│   ├── prod                    |  Different inventories for test & prod
│   └── test                    |
├── network-checkpoint.retry
├── network-checkpoint.yml
├── network-dmvpn-check.yml
├── network-dmvpn.yml
├── network-system.yml
└── roles                       |
    ├── cloudbuilder            |  Roles linked in as submodules.  Unit testing
    ├── network-backup          |  is performed on each role.
    └── network-dmvpn           |
```

Playbooks should embody the intent, architecture, and policy of a particular network.  It should *not* contain references to specific nodes nor the specific values that are used to configure these nodes.  That is:

#### Implementation (Inventory) + Definition (Playbooks}) = Deployment

This is the key capability that we need to test playbooks.  We architect the playbook to meet the needs of our production network.  We then create a testbed that mimics the key aspects of the production network, but at a smaller scale.  When the inventory for the test network is fed into the playbooks, it yields the test network.  This obviously gives us the ability to catch simple syntactical errors, but it also give the ability to test the architecture defined in the playbooks on something other than the production network.

#### Repository Types

For this session we are using 3 different types of repositories:

* Project Repository:  This is the top level repository that contains all of the Ansible Playbooks, Roles, and Inventory that is needed to a particular project.  It is representative of the kind of structure you could use in a production environment.
* Role Repositories: These are repositories linked that contain all of the playbooks, modules, vars, etc. that comprise a single Ansible Role.
* Inventory Repositories: These are repositories that contain the inventories that will be used in a partucular project.

#### Repository Modularity

To provide modularity, we have one main repository that represents a project, then several repositories linked in as submodules to provide the Roles.  Using submodules allows for keeping the roles in their own repositories to facilitate unite testing and the linking to a specific version of a Role.  Linking to a specific version of a Role that is known to work with a project allows for the choice of when to use a newer version.  This would generally be done as part of the integration testing done at the project repository level.

In addition to the Ansible Role repositories, we have repositories for the inventories.  Since the version   

### Included repositories and Roles

We include 3 Ansible Roles in the repository for demonstration:

* [network-dmvpn](https://github.com/ismc/ansible-network-dmvpn.git):  An Ansible Roles that deployes a DMVPN overlay over a hub and spoke network.
* [network-backup](https://github.com/ismc/ansible-network-backup.git): An Ansible Role that provides backup, checkpoint, and rollback for network devices.
* [cloudbuilder](https://github.com/ismc/ansible-cloudbuilder.git): An Ansible Role to build a cloud-agnostic model in a Public Cloud.

## Test Automation: Jenkins

### Testing Methodologies: Unit vs. Integration

#### Unit Testing

Unit testing in generally done in Ansible by testing roles.  An Ansible [Roles](http://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse.html) is the main unit of code reuse and portability.  Roles are an aggregation of tasks, modules, vars, etc. that accomplish a complex operation.  The are written once, then leveraged multiple times across different playbooks.  Unit testing is done at the role level.  By testing the Role, we validate that it properly undertakes specific the task for which it was written.  For the purposes of this session, that Role is [network-dmvpn](https://github.com/ismc/ansible-network-dmvpn.git)

#### Integration Testing

Integration testing is done when we integrate the Roles (i.e. the units) into the playbooks that setup the overall system.  For example, for DMVPN to work, the interfaces need to be properly configured.  This is a simple example, but there are several different systems of the network device (e.g. AAA, port configuration, routing protocols, etc.) are configured by different playbooks and they could all be tested together whenever possible.

### Testing Infrastructure

Since it is desirable to avoid testing on a production network whenever possible, a testbed is needed.  Generally, there are 2 options available:

* Physical: A representative, but scaled down version of the production network.
* Virtual:  This can take the form of a proper emulator like [Cisco VIRL](http://virl.cisco.com/) or by using VNFs on a virtualized infrastructure (e.g. OpenStack, VMware) or public cloud (e.g. AWS, Azure, GCE).

For this session, we use the example of a hub/spoke network that uses DMVPN as an overlay because it is regular (i.e. easy to automate) and easy to simulate.  In this case, our physical/production network if comprised of Cisco ISRs running IOS-XE.  Since Cisco IOS-XE is available in the form of the Cisco Cloud Service Router VNFs, we can create an accurate testbed using a public cloud.

### Test network

Our DMVPN testbed consists of 3 sites, each a VPC in AWS.  Each site have a Cisco CSR as its site router with and inside and outside interface.  Each inside network has a single host for connectivity testing:

![wan-testbed](wan-testbed.png)



## Usage

Since this is a complex system composes of several parts, all of those parts must be in places an configured to completely reproduce this work.  However, Since this repository used submodules, it has to be checked out recursivley:

```
https://github.com/ismc/devnet-2076_clus2018.git --recursive
```

To build the testbed:

```
ansible-playbook build-testbed.yml
```

To run the DMVPN Integration tests:

* Set the baseline system and interface configuration:

```
    ansible-playbook -i inventory/test network-system.yml
```

* (Optional) Checkpoint the routers after the initial configuration and before the DMVPN deployment.

```
    ansible-playbook -i inventory/test network-checkpoint.yml
```

* Deploy the DMVPN overlay:

```
    ansible-playbook -i inventory/test network-dmvpn.yml
```

* Check to make sure that DMVPN overlay as properly deployed

```
    ansible-playbook -i inventory/test network-dmvpn-check.yml
```

* (Optional) Rollback the router configuration to the state before the DMVPN overlay was deployed.

```
    ansible-playbook -i inventory/test network-rollback.yml
```

To destroy the testbed:

```
ansible-playbook desroy-testbed.yml
```

## License

GPL-3

## Author Information
* Steven Carter
* Chris Hocker
