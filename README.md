# DEVNET-2076: Cisco Live US 2018

This repository contains the companion material for the DEVNET-2076 session at Cisco Live US 2018.  It is a scaled down version of what might be used in a real environment that illustrates one possible DevOps pipeline for network automation with Ansible.  

## Background

### What is Infrastructure as Code?

Infrastructure as Code IaC can be summarized as defining your infrastructure as code, then maintaining the lifecycle of that code (and your infrastructure as a result) through an application development-like process.  This code is generally a task-by-task translation of the Method of Procedure (MOP) for a particular complex operation (e.g. Provision a new remote site, add a tenant network, etc).  More to the point, it is generally the culmination of the human experience of the Subject Matter Experts within the group that created the MOP.  Having these procedures and SME experience in code has the benefits of:

* **Configuration Management**: Ensure the correctness and consistency of the code that defines the network
* **Revision Control**: Manage and assign versions to the changes to the code
* **Drift Detection**: Detect configuration drift in the infrastructure by comparing it against the code that describes the infrastructure.
* **Communication**: Instead of the architecture and the experience to run it being locked in the head of a few, it is accessible by all.  Furthermore, it allows different SMEs to leverage other SME's area of expertise.

Another aspect of IaC is the ability to actually run the MOP after it is translated into code.  This gives someone the ability to undertake these operations in a rapid, repeatable way that reduces the number of variations in the configuration of a network.

## NetDevOps

### What is NetDevOps?

From Wikipedia:
> DevOps is a software engineering culture and practice that aims at unifying software
> development (Dev) and software operation (Ops). The main characteristic of the DevOps
> movement is to strongly advocate automation and monitoring at all steps of software
> construction, from integration, testing, releasing to deployment and infrastructure
> management.

That is, DevOps adds the process (e.g. collaboration, testing, deployment) that is needed to properly undertake automating with IaC.

## Why NetDevOps?

The ability to undertake operation fast with automation is enticing, but it must be done with caution and with focus on a particular outcome.  Automation tools such as Ansible are simply a mechanism to automate the individual tasks that are normally manually performed by a network operations teams.  Ansible has no innate intelligence for determining a good task from a bad task, so it will happily and efficiently create or destroy depending on the inventory and playbooks fed to it.  Also, Infrastructure as Code (IaC) network automation systems are generally made up of components from several different sources.  Each of these sources are separately developed and versioned.  A change in one of these components can perturb the entire system.  For these reasons, a NetDevOps process should be integrated into any meaningful IaC deployment to increase the likelihood of success and decrease any possible damage to operations.

What does that look like?

![netdevops](netdevops.png)

1. The engineer checks the branch (either personal or shared dev branch), makes the code change to that branch, and tests the change in local sandbox.  Once the engineer is satisfied with the change and has properly tested it, they create a Pull Request (PR) to tell everyone that they are ready to submit the change.  This is where others in the group get to see and comment on the change _before_ it is merged.  This is also were validation tests can be run _before_ the change is merged.

2. Jenkins, or some other automated testing engine, is used at this point to perform the validations tests.  These tests can be performed through Jenkins manually (either directly or something like a comment) or automatically for every PR.  In either case, Jenkins reports back the results of the tests to the ChatOps platform, Cisco WebEx Teams.

3. At this point, the group collaborates on the change to make sure that it is understood, that it aligns with organizational best practices, and that it is properly tested with standardized validation tests.

4. Once the change is agreed upon and properly tested, it is merged into the production branch.  

5. Jenkins can either test the production branch once the change is merged, at regular intervals, or both.

6. The change now lives in the production (i.e. "Golden") artifact repository.  From here, an orchestrion tool such as Ansible Tower can pick up that artifact and push it out as part of an operator-initiated request, a timed running to validate the configuration of the infrastructure with the IaC that defines it, or an API call from another application.

**In a strict IaC paradign, _EVERY_ change goes through this process.**

## Outcomes: Automated Humans vs. Automated Business

### Automating Humans

Let us pause for a moment to ponder what we are trying to achieve.  If we _just_ want a human to be able to perform an operation faster by automating all tasks with IaC within a NetDevOps workflow, then we might not _actually_ achieve that.  Why is this?  Automation is geared at deploying single or multiple changes across a large number of devices.  However, many changes do not fall into this category.

### Engineering vs. CRUD

There are basically two types of changes that are made to a network in steady state operations:

* Architectural/Engineering: These are changes to the architecture of the network (e.g. Routing, QoS, Multicast) that generally affect the entire network.  It is also the architecture for how new services are deployed (e.g. tenants, remote sites, etc.)
* Create, Read, Update, Delete (CRUD): These are changes that deal with delivering of network services to a particular customer or application (e.g. Putting a port in a VLAN, adding an ACE to an ACL, or adding a load balancing rule).

The rigor of IaC is absolutely the way to make major architectural changes to a network because of the network-wide effect that these changes have and the relatively small number of changes that occur.   

CRUD is often different, however.  Making a single change (e.g. SNMP Community Strings) on thousands of devices is an operation for which the overhead of NetDevOps is justified.  However, if a single change is needed on a single device (e.g. Change a interface's access VLAN), the NetDevOps overhead slows that task down significantly.

This overhead is not necessarily a bad thing, even for small changes.  There are significant advantages in enforcing configuration management, revision control, code review, and testing on _every_ change.  It does not, however, always make network operations faster or easier.  This increased friction can make it unpalatable to many network teams and hinder adoption.

We should then focus more on automating business processes and less on _just_ automating humans.  

### Source of Truth (SoT)

One area in which Devops for application development differs from DevOps for network operation is the Source of Truth.  The Source of Truth of a network contains all of the values that make a network _that_ network.  In its simplest form, those values are things like hostname, NTP servers, users, AAA servers, etc.  These values are relatively few in number and are used across all of the devices.  The problem comes with values that define things like interface configuration, ACLs, and load-balancing rules.  Even for a mid-size network, the number of values that define the configuration of that network could be over 100,000.  Keeping 100,000+ values in flat files in a code repository can be at best painful and at worst unmanageable.

Another use case in which the separation of the SoT from the Code is critical is Cloud and other virtualized platforms.  On these virtualized platforms, virtual routers, firewalls, load-balancers, etc. are being dynamically created and destroyed to accomodate customer requests and/or user workloads.  To manually copy this inventory data from their native platforms (e.g. AWS) to flat files would fight against the agile posture that we are trying to achieve with automation.  This can better be accommodated by pulling the SoT dynamically directly from those platforms.

This is why, for any meaningful deployments, the SoT is kept and managed separately from the code that references it.  This bifurcation helps address both the overhead problem and, as we'll see later, testing.

### Implementation (SoT) + Definition (Code) = Deployment

When we decouple the SoT from the code, it is the code that is strictly managed through the full NetDevOps workflow since changes to it affect the architecture and policy of the entire network, which justifies the associated rigor and overhead.

The SoT is then managed separately.  This is not to say that none of the SoT is in code/file form.  Much of the common values that define constants across the network and have a large collateral affect (e.g. NTP servers, AAA servers, SNMP Community Strings) can still be managed with code.  But the values associated with the day-to-day CRUD generally live in an external database or CMDB.

### Automating Business Processes: API-Driven Automation

This SoT/Code bifurcation also enables the automation of the business processes that were likely the original goal of the enterprise's automation effort in the first place.  When the SoT is external to the automation infrastructure, the values that comprise it can be changed externally, then feed into the process.  For example, a user that wants to add an exception for a server can go to a self service portal to request that exception.  That request can then go through the review process to make sure that it is aligned with business policies and appropriately approved.  The SoT can be updated with this exception and the portal can call an API to tell the automation infrastructure to push out that change.  This accomplishes two equally important objectives:

1. It takes the network team out of the CRUD
2. Is allows the network team to define and put checks around how changes to the network are performed

## DevOps Infrastructure

For this session, we present an infrastructure to implement a full NetDevOps workflow in a way that is representative to what many organizations deploy.  It is, in fact, a real-world deployment in that it is the "Dev" part of a set of companion sessions that we are delivering at CLUS 2018.  The other Session, BRK-2023: Building Hybrid Clouds in Amazon Web Services with the CSR 1000v, includes a demo for deploying CSR-based cloud nodes in public clouds and connecting them with a DMPVN overlay.  That demo uses the production artifacts created in this session.

Our NetDevOps workflow is facilitated by the following infrastructure:

* Source Code Management: GitHib
* Test Automation: Jenkins
* Production Deployment: RedHat Ansible Tower
* Collaboration Platform: Cisco WebEx Teams

### Source Code Management: GitHub

We use GitHub for our source code management because of its ubiquitous nature and easy access.  Many organizations leverage GitHub Enterprise or some other internally install git system like GitLab or BitBucket.  Most SCMs have similar mechanisms to what is laid out in this description.

#### Repository Layout

The layout of the repo is pretty standard for Ansible.  The main point is in how the layout leverages the default behavior of Ansible and facilitates the NetDevOps workflow.  For example, Ansible allows one to specify the inventory to be used.  In this repo, we have two different inventories: test and prod. A couple of key points to remember about Ansible is that an `inventory` contains both the list of devices to automate and the key/value pairs (a.k.a Source of Truth) that defines how those devices are configured.  

```
.
├── Jenkinsfile
├── README.md
├── ansible.cfg
├── build-cloud.yml
├── check-ssh.yml
├── destroy-cloud.yml
├── inventory                   |
│   ├── prod                    |  Different inventories for test & prod
│   └── test                    |
├── network-backup.yml
├── network-checkpoint.yml
├── network-command.yml
├── network-dmvpn-check.yml
├── network-dmvpn.yml
├── network-rollback.yml
├── network-system.yml
└── roles                       |
│   ├── ansible-tower-api       |  Roles linked in as submodules.  Unit testing
│   ├── cloudbuilder            |
│   ├── network-backup          |
│   └── network-dmvpn           |
└── update-tower.yml
```

Playbooks should embody the intent, architecture, and policy of a particular network.  It should *not* contain references to specific nodes nor the specific values that are used to configure these nodes.  That brings us back to:

#### Implementation (SoT) + Definition (Playbooks) = Deployment

This is the key capability that we need to test playbooks.  We architect the playbooks to meet the needs of our production network (i.e. architecture and policy).  We then create a testbed that mimics the key aspects of the production network, but at a smaller scale.  When the inventory/SoT for the test network is fed into the playbooks, it yields the test network.  This obviously gives us the ability to catch simple syntactical errors, but it also give the ability to test the architecture defined in the playbooks on something other than the production network.

#### Repository Modularity

To provide modularity, we have one main repository that represents a project, then several repositories linked in as submodules to provide the Roles.  Using submodules allows for keeping the Ansible Roles in their own repositories to facilitate unit testing and the linking to a specific version of that repository.  Linking to a specific version of a Role that is known to work with a project allows for the choice of when to use a newer version.  This would generally be done as part of the integration testing done at the project repository level.

#### Repositories

In this example, we are using 3 different types of repositories:

* **Role Repositories**: These repositories are linked in as submodules and contain all of the playbooks, modules, vars, etc. that comprise the individual Ansible Roles:
  * [network-dmvpn](https://github.com/ismc/ansible-network-dmvpn.git):  An Ansible Role that deployes a DMVPN overlay over a hub and spoke network.
  * [network-backup](https://github.com/ismc/ansible-network-backup.git): An Ansible Role that provides backup, checkpoint, and rollback for network devices.
  * [cloudbuilder](https://github.com/ismc/ansible-cloudbuilder.git): An Ansible Role to build a cloud-agnostic model in a Public Cloud.
* **Inventory Repositories**: These repositories contain the inventories that will be used by a particular project.
* **Project Repository**:  This is the top level repository that contains all of the Ansible Playbooks, Roles, and Inventory used in our [ARCBRK-2023](https://github.com/ismc/brkarc-2023_clus2018) session.  It is representative of the kind of structure you could use in a production environment.

### Branch protection

GitHub's [branch protection](https://help.github.com/articles/about-protected-branches/) ensures that collaborators on your repository cannot make irrevocable changes to branches. Enabling protected branches also allows you to enable other optional checks and requirements, like required status checks and required reviews.

For this demonstration, we protect the `master` branch and develop on the `devel` branch.  Code changes can be checked in directly to the `devel` branch.  The `devel` branch is tested ad-hoc and at regular intervals.  Changes are not allowed to be pushed directly into the `master` branch and must go through the full NetDevOps workflow by using a Pull Request (PR).  The `master` branch is also tested ad-hoc and on regular intervals.

### Test Automation: Jenkins

Jenkins is a self-contained, open source automation server which can be used to automate all sorts of tasks related to building, testing, and delivering or deploying software.

#### What kind of changes can we test?

To use somewhat circular reasoning, we can test anything for which we can build a test.  Testing the syntactical correctness is easy.  We can simply run the playbooks against anything and they will either fail or not.  Better yet, we can use Ansible's built-in syntactical checker.  To test the functionality of the code, we perform a combination of Unit and Integration testing.

For most everything else, a validation test much be concocted to for a particular change.  That test can either be focused on that one partucular test, or a system test to make sure that system is operating as a whole.  Advertising a new network via BGP, for example, is a use can that can be tested directly.  In this case, the network can be added to the prefix list, then that prefix list can be checked in the list of routes being advertised by the router.

In this session, we are deploying a DMVPN overlay.  We could validate each component of the DVMPN deployment (e.g. The tunnels are up, the neighbors are seen via NHRP, the EIGRP routes are present, etc.), but we perform a system test instead.  That system test pings every router's tunnel interface from every other router, validating the tunnels and HNRP functionality, then it pings each router's loopback address from every other router, validating EIGRP functionality.

#### Testing Methodologies: Unit vs. Integration

#### Unit Testing

Unit testing in generally done in Ansible by testing roles.  An Ansible [Role](http://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse.html) is the main unit of code reuse and portability.  Roles are an aggregation of tasks, modules, vars, etc. that accomplish a complex operation.  They are written once, then leveraged multiple times across different playbooks.  Unit testing is done at the role level.  By testing a Role, we validate that it properly undertakes the specific task for which it was written.  For the purposes of this session, that Role is [network-dmvpn](https://github.com/ismc/ansible-network-dmvpn.git)

#### Integration Testing

Integration testing is done when we integrate the Roles (i.e. the units) into the playbooks that setup the overall system.  For example, for DMVPN to work, the interfaces need to be properly configured.  This is a simple example, but there are several different systems on a network device (e.g. AAA, port configuration, routing protocols, etc.) that are configured by different playbooks and they should all be tested together whenever possible.  For this session, integration testing is done by testing the main BRKARC-2023 repository that integrates the Roles together.

### Testing Infrastructure

Since it is desirable to avoid testing on a production network whenever possible, a testbed is needed.  Generally, there are 2 options available:

* Physical: A representative, but scaled down version of the production network.
* Virtual:  This can take the form of a proper emulator like [Cisco VIRL](http://virl.cisco.com/) or by using VNFs on a virtualized infrastructure (e.g. OpenStack, VMware) or public cloud (e.g. AWS, Azure, GCE).

### Test network

For this session, our production network is a multi-site cloud network with a physical datacenter.  The playbooks for that operation rely on two roles: cloudbuilder to build the cloud nodes and network-dmvpn to configure the DMPVN overlay that connects the sites together.  We chose these scenario because it is a common deployment and it is an easy deployment for which to crate a test network.  Since the spokes are virtualized in a Public Cloud, we can simple virtualize them the same way in the test network.  The physical site router runs IOS-XE, so we can visualize that using the same Cisco Cloud Services Router that we use in the spokes.  This gives us a very accurate approximation of our production network for our testing.

Specifically, our DMVPN testbed consists of 3 sites, each a VPC in AWS.  Each site has a Cisco Cloud Services Router as its site router with and inside and outside interface.  Each inside network has a single host for connectivity testing:

![wan-testbed](wan-testbed.png)

### Production Deployment: Ansible Tower

Ansible Tower as a means for pushing out production artifacts is important for 2 reasons:

**Controlled release of automation**:  If you go through all of the work of making sure that your code is correct, you also want to make sure that it is run on the right devices, by the right people, at the right times.

**API-Driven Automation**: In order for the network team to get out of the CRUD, we need to provide access to other application in the enterprise.  Most often, this external application is a self-service portal like Service Now.  Generally, these self-service portals are also configuration database that can act as part of the SoT.

Putting all of this together, we get the workflow depicted in the following diagram.  The left side of the diagram shows the "Dev" part DevOps that we have already covered.  The right side of the picture is where the "Ops" comes in and one way that it can be used in automating a business process.  In this case, we use Ansible Tower to push out the production "artifacts".  These artifacts can be push out in one of three ways:

* **Timed Running**: This is a regular running of playbooks to detect configuration drift or to push out new services.
* **Operator UI**: Ansible Tower provides a more approachable interface to the automation infrastructure that can enforce roles-based access, manage credentials, and log activity.
* **API-Driven Automation**: The last, and potentially most powerful form of leveraging the production playbooks created in the Dev process is the API.  One potential workflow illustrated in the figure is:

  1. User requests new service (e.g. A new public cloud node)
  2. After going through the approval process, Service Now calls the Ansible Tower API to create the new service
  3. Ansible Tower runs the playbook, feeding it the appropriate inventory and SoT information to create the service.
  4. Ansible Tower calls the Service Now API to append information to the request and close the ticket, informing the user
  5. The service is now available to the user for consumption.

  This workflow can be used to decommission the service as well.

![automated_enterprise](automated_enterprise.png)

### Collaboration Platform: Cisco WebEx Teams

ChatOps functions are achieved using WebEx Teams.  The WebEx Teams app provides many integrations with various DevOps tools, including both GitHub and Jenkins.  These can be enabled for an arbitrary space via the WebEx Teams App Hub.

### Jenkins
In order to enable the Jenkins integration, you need to do the following:

* Visit the WebEx Teams App Hub and enable the Jenkins integration for your space and copy the created webhook URL
* Install the Notification plugin on your Jenkins server
* Add an entry in your Jenkinsfile to call the WebEx Teams webhook you copied earlier for all notifications

The required entry in the Jenkinsfile can be found by using the "Pipeline Syntax" functionality in Jenkins.  Go to the pipeline page in the Jenkins server and select Pipeline Syntax, then "properties" from the drop down list.  In here you can insert the webhook URL and generate the needed script code for your Jenkinsfile.

The properties section should look something like:
```
properties([[$class: 'HudsonNotificationProperty', endpoints: [[buildNotes: '', urlInfo: [urlOrId: 'http://cisco-spark-integration-management-ext.cloudhub.io/api/hooks/8fde6043-69b6-11e8-bf37-0123456789ef', urlType: 'PUBLIC']]]]])
```

## License

GPL-3

## Author Information
* Steven Carter
* Chris Hocker
* Jason King
