Towards TOSCA 2.0
=================

Draft 2 by Tal Liron

## What Is Needed

Our lessons from implementing and using TOSCA 1.X have revealed some major deficiencies in its design:

### 1. Services and Plugins

Cloud topologies are not isolated abstractions, but always inherently integrated with the tooling used to make them a reality. Users need a way to configure, invoke, and otherwise interact with their tools. For example: credentials and settings for OpenStack APIs, installation and execution of Chef recipes, registering with management/discovery services, responding to triggers from monitoring/alert systems, etc.

TOSCA 1.X allows users to provide hints to services via typed "policies." However, "policies" are just static bundles of properties, and the expectation is that external tools would know what to do with them. What is needed is a two-way membrane, a grammar that allows users to specify tooling, as well as use functionality exposed by the tools.

We will here propose to replace "policies" with what we call "services", implemented as *plugins* that allow comprehensive integration between topology and tools while maintaining a high level of abstraction. For example, a generic "Provisioner" service can provision compute nodes, but can be *implemented* by OpenStack, AWS, or Docker services, which in turn can be written in JavaScript for one orchestrator (Puccini) or Java for another, and indeed there may be several implementations per orchestrator. Users of a specific orchestrator can decide which to use, via configuration or UI switches, but topology authors do *not* have to target a specific orchestrator. Furthermore, code for custom services can easily be bundled into a CSAR.

### 2. Provisioning and Runtime Requirements

Astonishingly, despite being cloud-centric, TOSCA 1.X does not address the most critical and problematic phase in deploying cloud services: *provisioning*. TOSCA 1.X assumes that the orchestrator will create all the *explicitly* described virtual machines and containers during the "install" phase. This is a bizarrely idealistic expectation. Part of the problem is the requirements-and-capabilities engine: despite being a powerful shortcut for design-time, it does not play a role in runtime for TOSCA 1.X.

We will here propose that requirements do not have to be explicitly satisfied by explicit nodes. Rather, they can be *implicitly* satisfied by a plugin service during a provisioning phase. For example, a MySQL software node might request a virtual machine with at least 4 GB of RAM and NUMA paging enabled, even such a node was not explicitly declared in the topology. The allocator service will provision as many such machines as necessary and add them to the topology.

The same mechanism can be used to satisfy all kinds of runtime requirements: Need an expensive VNF to complete an NFV service chain? It can be licensed and allocated on the spot during provisioning. Need an HTTP loadbalancer to scale a deployed REST API service? A "LoadBalancing" service can automatically add one, configure it to proxy to the HTTP nodes, open internal firewall ports, etc.

This new provisioning phase requires us to rework the normative lifecycle. Having an *install* operation in a node, as in TOSCA 1.X, no longer makes sense if its compute node has not yet been provisioned. So, we propose moving most of the normative lifecycle operations to *relationship*s. For example, our MySQL software will be installed in the *add\_target* phase of its *Installed* relationship to the provisioned virtual machine.

(Note that this approach is inspired in some ways by Juju. The main difference is that Juju doesn't have an overall plan for the deployment, which is exactly what TOSCA provides.)

### 3. Operations and Plugins

Interfaces, operations, and inputs are defined far too thinly in TOSCA 1.X. When you are working with a robust service (OpenStack, Chef, Juju, etc.) it's difficult, error-prone, and a security risk to dump all the work into a custom Bash script. And even Bash scripts aren't so straightforward: What if you need to run them remotely over SSH? Or sandboxed locally?

We will here propose that "operations" not be merely paths to Bash scripts, but instead be first-class entities in TOSCA. Services (plugins) can then expose robust operation types that have been thoroughly tested and are ready for production environment. Different orchestrators can provide their own custom implementations for normative operation types.

### 4. Grammar Overhaul

The object-oriented architecture has proven to be handicap in TOSCA 1.X. Specifically, deriving types via single inheritance is far too inflexible and limiting. The primary victim is the ability to create multi-cloud service templates: if you need to derive *OpenStackCompute* and *AwsCompute* from *tosca.nodes.Compute*, but also need to derive your own special host from *tosca.nodes.Compute*, then you're stuck picking one *and only one* cloud platform as your parent type. This is one of the biggest obstacles for using TOSCA in NFV, where arguments over which awkward workaround to use have become debilitating distractions.

More generally, users often want to combine more than one piece of reusable functionality into a node. They also need more dynamic ways to do so: the ability to choose during instantiation, for example, between the OpenStack functionality of a secure network or its AWS functionality.

Some of this work could be done in the existing TOSCA 1.X paradigm by moving all functionality to "capabilities" within the nodes instead of nodes themselves, and treating nodes as assemblages of capabilities. But it would be much better if the grammar were designed from the ground up around such dynamic assemblages. That is what we will propose here. So, we're saying goodbye to object-oriented architecture and and introducing paradigms that are much more appropriate for complex cloud topologies.

### 5. Unified Syntax

TOSCA 1.X has an unnecessary extra level of abstraction. Consider the move from node type, to node template, to node instance. The syntactical differences between types and templates mean that often simple things, such as adding a property, require users to create *both* kinds of entities. This is confusing, verbose, and unnecessary.

Furthermore, TOSCA 1.X is burdened with *eight* different kinds of type hierarchies: node, capability, relationship, group, interface, policy, data, and artifact. Each has a different syntax, with idiosyncratic and hard-to-follow (and hard-to-understand) rules for object-oriented inheritance and overriding. For example: a node type can define a relationship internally, and that relationship can define an interface of a certain type, which in turn can override operation inputs. What is the relation to the interface type, relationship type, and parent node type?

We will here suggest collapsing the concepts of "type" and "template" to a single "entity" that satisfies all the grammatical requirements with a single, coherent syntax, and that we furthermore remove the distinctions between different types of "types." This unified syntax will allow us to vastly simplify TOSCA parsers as well as making TOSCA much easier to learn.

## Further Reading

* [Learn by Example](example.md)
* [Specification](specification.md)
