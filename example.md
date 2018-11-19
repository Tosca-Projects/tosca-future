Learn by Example: Node Cellar
=============================

Let's create a simple enterprise cloud service: [Node Cellar](http://nodecellar.coenraets.org/), a Node.js/MongoDB web application that can be deployed on various cloud platforms.

[Source](nodecellar.yaml)

First, notice the *tosca\_grammar* directive, and that we are *explicitly* importing the normative profile. Unlike TOSCA 1.X, we are separating the grammar from the profile of normative entities, and are *not *implicitly including the profile. Indeed, we expect the grammar and profile to each have their own spec, with its own versioning.

This section, *assign*, is where we put entities into our service template. It complements *declare*, where entities are put for general use. This paradigm replaces the "type" vs. "template" distinction in TOSCA 1.X, and indeed you'll see it used throughout TOSCA 2.0.

Under *assign* or *declare* we have named entities. Entities include nodes, relationships, services, groups, policies, etc. Note that all of these have the same syntax. Also note that there is no difference in syntax between assigned and declared entities. (However, there is a difference between assigned and declared properties; more on that later.)

In the entity's *imply* section we provide a list of names of other entities that are now *part* of this entity. This is explicitly *not* object-oriented inheritance, but instead more akin to a [mixin](https://en.wikipedia.org/wiki/Mixin): all properties of the implied entities are now part of *this* entity. In turn, if those entities imply other entities, they also become part of this one.

An important feature is that it is possible to imply the same entity more than once, as we do here for the public and admin endpoints. If you do so, you *must* give each entity a unique alias so that it will be possible to reference the properties individually. The parser will generally make sure that there is no overlap in names, and if it encounters ambiguity will emit an error informing you that you should use an alias.

The *assign* section in the entity is where we assign values to properties. Each property must have been already declared in a declare section. (We don't see a declare section in this case, because we received all the declaration from our implied entities, but we'll examine declare in detail later.)

This is where the names, namespaces, and aliases matter. There are various ways to specify a property:

1. The fully qualified name, with the namespace and entity name as prefix: *tosca.Node.instances*. Note that *tosca.Node* is implied by *tosca.Software*.
2. If the namespace was imported, we can omit it: *Node.instances*.
3. If there is no ambiguity of properties names within this entity, we can omit the entity, too, as we do here: *instances*

Also note that the "." is not arbitrary, and is used for namespace qualifiers. It is possible to use two different styles to reference a name, either using "." or cascading YAML dicts:

    Node.instances: 1

    Node:
      instances: 1

It is also possible to mix the two styles.

Here we're assigning values to properties with the *tosca.endpoint.Http* entities using the aliases we provided above.

You might be asking: What kind of entities are these endpoints? Are they data types? Capabilities? Or just properties within the node? The answer is: it doesn't and *shouldn't* matter. All entities work the same way in TOSCA 2.0: they can all declare and assign properties, imply other entities, etc.

You'll also see the use of function calls here, which essentially is identical to TOSCA 1.X. The difference is that we are requiring the use of a "!" prefix for all function names. This makes sure that there are no overlaps in names and clarifies that this indeed a function call. (TOSCA 2.0 lets you define your own functions, too!)

There also no need to have separate *get\_input*, *get\_property*, and *get\_attribute* functions in TOSCA 2.0. Indeed, there's nothing special about Inputs: it's just a normal entity that we added to our service template, which you can see defined below.

The *require* section is where things get really powerful. This is where we create relationships to other entities. At the minimum we name the entity we want, though note that the requirement can be satisfied by any entity that *implies* our specified entity. So, there's no real difference here between requiring a "node" vs. requiring a "capability": the "capability" might be part of another node, or might not be. (It might not be part of our topology at all! More on that later.)

The *relate* section is optional, and allows us to specify an entity that will be instantiated as an edge between our entity and the required entity. (Like TOSCA 1.X, it is possible to have an "empty" relationship with another entity.) As you may expect by now, there's nothing special about relationship entities. Again, they have properties that can be declared and assigned.

In this case we are using a *tosca.InstalledOn* entity, which declares *source* and *location* properties. But let's take a closer look at the *configure* property: it is, in fact, declared as an operation type.

There is no separate interfaces section in TOSCA 2.0! Operations are first-class citizens, and declared/assigned like any other property.

When and how are these operations called? Well, just like TOSCA 1.X, they are part of a workflow. The *tosca.Install* workflow (also just another entity) will iterate all the *tosca.InstalledOn.configure* properties and call them in the correct order. (There is also a *tosca.InstalledOn.install* declared in *tosca.InstalledOn*, but it's already assigned a value there and we don't need to override it here.)

As for the assigned value of configure, you'll notice the "!" prefix indicating that we are calling a function. In this case, the function is *an entity name*. Such a function is called a "constructor", and it's used to create values of a specific type. The *configure* property is declared to be of type *operation*, and here we're assigning a *tosca.ExecuteAt* entity to it. As long as *tosca.ExecuteAt* implies *operation*, which it does, this will be valid. As you might guess, *ExecuteAt* is used to run executables, such as scripts, on a remote host. We must specify the artifact, and can optionally send arguments to it. By default it will look travel the topology graph to find a *tosca.Compute* node, but it can be configured to run on a specific location.

Our next entity, *NodeJs*, is simpler. However, it does introduce a new feature in its *require* section: *assign*. Here we specify the property values we expect the required entity to have. As with *node\_filter* in TOSCA 1.X we can use constraints here. However, unlike TOSCA 1.X, constraints don't require their own specialized syntax: there are simply functions. Again, users can create their own custom functions for use in sophisticated constraint specification.

*MongoDb* is similar to *NodeJs*, so let's skip it.

Our *Install* entity is a workflow. Note that normally it's unnecessary to explicitly assign all workflows to our service template: the orchestrator will automatically add the default ones. However, we might want to customize them, as we do here. Our version of the *Install* entity implies the default one, but also declares and assigns a *url* property. This is how we handle *outputs* in TOSCA 2.0.

You might have noticed that both MongoDb and NodeJs both require a tosca.Compute entity, but we have assigned none. If this was TOSCA 1.X, this service template would fail validation, because requirements-and-capabilities matching runs during the instantiation phase, which happens before the install workflow.

However, this is where the last line comes in:

This curious line is actually just a syntactical shortcut for this:

    Provisioner:
      imply:
      - tosca.Provisioner

This allocator is a *service*. In particular what this service does is hook a function to the requirements-matching phase. If a requirement is not met, it then has the ability to dynamically modify the topology by adding new entities. In this case it will provision *tosca.Compute* nodes for us, if it can, according the requirements.

Obviously it can't be this simple. Will these compute nodes be virtual machines or containers? Will they be on OpenStack, AWS, or Docker? Where and under what credentials? Will a separate compute node be created for each piece of software, or will we allow a single compute node to support several software installs? All of this could technically be configured via properties. However, obviously we can't declare all possible properties in *tosca.Provisioner* for all cloud platforms. And what about support for cloud platforms that don't yet exist?

This is solved via a powerful feature in TOSCA 2.0: *replacement*. Here's how the *tosca.Provisioner* entity is declared in the normative profile import:

TODO

We also have entities for each cloud platform:

The *replace* section is grammatically identical to *imply*: the replaced entities are contained in ours. However, replace also means that our entity can work *instead* of these entities. When the orchestrator sees our reference to *tosca.Provisioner* in the NodeCellar service template, it will know that it has at least three entities that can replace it. Because that is an ambiguity, it will emit an error, telling us that we need user intervention to choose which one we want. The orchestrator can let us choose via its UI, or via stored configuration. If there's no ambiguity, then the replaced entity will work instead of the once we specified. So, for example, if we chose *tosca.openstack.Provisioner*, our service template would work *as if* we did this:

    Provisioner:
      imply:
      - tosca.openstack.Provisioner

Voila, we will be able to use the same service template with either OpenStack or AWS.

A replacing entity can be further replaced by yet another entity. Indeed, this is how plugins can work to extend an already existing service template, or even extend other plugins.

Furthermore, it's how the platform itself works! For example, Puccini might replace the abstract normative *tosca.openstack.Provisioner* like so:

TODO

The same technique is used to replace the default workflows:

TODO

And even operation types:

TODO

To keep thing simple, this short NodeCellar example has not covered every feature. Here are some extra details:

- **Attributes**. TOSCA 1.X has separate sections for properties and attributes. In TOSCA 2.X there are just properties, except that you can declare them with *mutable: true*, effectively turning them into attributes.
- **Groups**. We know that *imply* (and *replace*) can be used to contain another entity in ours, but a group is different: actually, it should *refer* to those entities. The solution is actually very trivial: a group is an entity that uses *require* for other entities (which can in turn also be groups) using the *tosca.RefersTo* relationship. That's it!
- **Combining operations**. What if we want our *tosca.RelateTo.add\_target* operation to do several things, for example installing several packages and then running a custom script? Not a problem: use the *tosca.Serialize* entity to execute several operation in sequence, or *tosca.Parallelize* to execute them concurrently, and nest either or both types to any depth.
- **Expression language.** Constraints in TOSCA 1.X only allow for a logical "and" between clauses. TOSCA 2.0 introduces more flexible expressions: "and", "or", "not" and parenthesizing.
