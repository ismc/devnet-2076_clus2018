# DEVNET-2076: Cisco Live US 2018

This repo contains the material used in the DEVNET-2076 session at Cisco Live US 2018.  It is a scaled down version of what might be used in a real environment that illustrates one way of doing CI/CD for networks with Ansible.

## Background

The layout of the repo is pretty standard for Ansible.  The main point is in how the layout is used.  For example, Ansible allows one to specify the inventory to be used.  In this repo, we have two different inventories: test and prod. A couple of key points to remember about Ansible is that an `inventory` contains both the list of devices to automation and the key/value pairs (a.k.a Source of Truth) that defines how those devices are configured.  Second, playbooks should embody the intent, architecture, and policy of a particular network.  It should *not* contain references to specific nodes nor the specific values that are used to configure these nodes.

#### Implementation (Inventory) -> Playbooks (Architecture) -> Deployment

This is the key capability that we need to test playbooks.  We architect the playbook to meet the needs of our production network.  We then create a testbed that mimicks the key aspects of the production network, but at a smaller scale.  When the inventory for the test network is fed into the playbooks, it yields the test network.  This obviously gives us the ability to catch simple syntactical errors, but it also give the ability to test the architecture defined in the playbooks on something other than the production network.

For this session, we use the example of a hub/spoke network that uses DMVPN as an overlay because it is regular (i.e. easy to automate) and easy to simulate.

## Testing Methodologies: Unit vs. Integration

### Unit Testing

Unit testing in generally done in Ansible by testing roles.  An Ansible [Roles](http://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse.html) is the main unit of code reuse and portability.  Roles are an aggregation of tasks, modules, vars, etc. that accomplish a complex operation.  The are written once, then leveraged multiple times across different playbooks.  Unit testing is done at the role level.  By testing the Role, we validate that it properly undertakes specific the task for which it was written.  For the purposes of this session, that Role is [network-dmvpn](https://github.com/ismc/ansible-network-dmvpn.git)

### Integration Testing

Requirements
------------

Usage
--------

License
-------

GPL-3
