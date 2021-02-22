---
title: "Dependency Namespaces and Labels"
number: "003"
status: "provisional"
owner: "@carolynvs"
authors:
  - "@carolynvs"
  - "@steder"
created: "2021-02-23"
type: "feature"
---

* [Abstract](#abstract)
* [Motivation](#motivation)
  * [Use Case: Exclusive Resource](#use-case-exclusive-resource)
  * [Use Case: Global Resource](#use-case-global-resource)
  * [Use Case: Scoped Resource](#use-case-scoped-resource)
  * [Use Case: User Provided Resource](#use-case-user-provided-resource)
  * [Use Case: Polymorphic Resource](#use-case-polymorphic-resource)
* [Rationale](#rationale)
* [Specification](#specification)
  * [Match Criteria](#match-criteria)
    * [Label Selector](#label-selector)
    * [Namespace Selector](#namespace-selector)
    * [Name Selector](#name-selector)
    * [Interface Selector](#interface-selector)
  * [Use Case Solutions](#use-case-solutions)
    * [Solution: Exclusive Resource](#solution-exclusive-resource)
    * [Solution: Global Resource](#solution-global-resource)
    * [Solution: Scoped Resource](#solution-scoped-resource)
    * [Solution: User Provided Resource](#solution-user-provided-resource)
    * [Solution: Polymorophic Resource](#solution-polymorphic-resource)
  * [Porter Bundle Index](#porter-bundle-index)
    * [List Indexed Bundles](#list-indexed-bundles)
    * [Register a Bundle](#register-a-bundle)
    * [Unregister a Bundle](#unregister-a-bundle)
  * [Post-Install Dependency Management](#post-install-dependency-management)
    * [Managed Dependency](#managed-dependency)
      * [Detecting Unused Managed Dependencies](#detecting-unused-managed-dependencies)
    * [Unmanaged Dependency](#unmanaged-dependency)
  * [Dependency Resolution](#dependency-resolution)
* [Implementation](#implementation)
* [Backwards Compatibility](#backwards-compatibility)
* [Security Implications](#security-implications)
* [Rejected Ideas](#rejected-ideas)
* [Open Questions](#open-questions)

## Abstract

Support advanced dependency management use cases, such as shared dependencies, using existing installations, and satisfying a dependency with an interface, through the use of namespaces and labels.

___

## Motivation

Initial Proposal: https://github.com/getporter/porter/discussions/1441

When we initially implemented dependencies in Porter, we knew that there were a lot of different directions that we could take but we were missing strong use cases and experience with what a dependency looked and how it behaved with bundles.
We implemented the most obvious scenario (wordpress requires its own mysql database) and decided to wait for further feedback before continuing.
Now that we have been working with bundles for a while a set of use cases have evolved:

### Use Case: Exclusive Resource

> I need my own instance of my dependency

When a bundle is run, new installations are created for its dependencies.
For example, an application requires a dedicated redis cache per installation.
This is the only supported use case by Porter today.


### Use Case: Global Resource

> I just need my dependency to be present

I am writing the Porter Operator to work with Operator Lifecycle Manager (OLM).
When I install the Porter Operator, OLM should be installed on the cluster.
It should not be installed more than once.


### Use Case: Scoped Resource

> My micro-services share a common dependency

My application has 4 micro-services and a shared postgres database.
When I install the first service, a database should be created for it.
When I install the other services, I want to share the database (and the connection string) between them.


### Use Case: User Provided Resource

> I already installed the dependency previously

I don't want Porter to install SQL Server for me, but I do want Porter to help connect the outputs from a named installation into another bundle.
For example, a manually set up on-premise shared database server, the 64 core beast that you keep in the office closet.
I want to automatically inject the connection string for the database into the bundles I install.

### Use Case: Polymorphic Resource

> I need a dependency that fulfills an interface, which multiple dependencies are capable of providing.

Consider the many different ways that the MySQL database can be installed and used.
It could be installed on your local cluster using Helm. You may have a local server running on your desktop. It could be installed on a virtual machine on your cloud provider. Every cloud provider offers a managed MySQL service. All of these databases have something in common: a specific supported MySQL version and a connection string.

I want my bundle to be able depend on a bundle that provisions a specific version of MySQL and outputs the connection string, without having to take a hard dependency on a specific implementation.

___

## Rationale

Bundle dependencies aren't exactly the same problem space as with software library or OS package dependencies.
The difference when dealing with instances of installed software is the added concept of ownership.
A bundle dependency needs to be able to express not only the name and version of the dependency but how the instance of that dependency's software may (or may not) be shared.

The resulting design you see in this proposal is a composite of traditional dependency management and Kubernetes' method of referencing resources using namespaces and label selectors.
Namespaces provide scope, allowing multiple instances of software to exist side-by-side, and label selectors give us a flexible language for declaring how to identify those instances when you may not have created them yourself.

___

## Specification

The heart of the solution for all of the above use cases is reducing them to a single decision: "GetOrCreate". The bundle declares a set of match criteria that is used to identify an existing installation that satisfies the dependency. If one cannot be resolved, then a new installation is created.

### Match Criteria

Currently, a dependency can be resolved via the bundle name and version. We can add additional criteria to give bundle authors and end-users more control over reusing an existing installation for a dependency:

* [Label Selector](#label-selector)
* [Namespace Selector](#namespace-selector)
* [Name Selector](#name-selector)
* [Interface Selector](#interface-selector)

#### Label Selector

Every installation can have a set of string labels defined, KEY=VALUE. 
When an existing installation cannot be reused, a new installation is created with the specified labels.


#### Namespace Selector

Installations can be scoped to a namespace, or the namespace can be left unset to indicate that it is a global installation.
When dependencies are resolved, Porter resolves the dependency using only the bundles in the global namespace and the current namespace if set.
When an existing installation cannot be reused, a new installation is created in the specified namespace.

#### Name Selector

When a user installs a bundle, they can tell Porter to use a specific named installation to satisfy a dependency.


#### Interface Selector

Bundles can declare that it implements a well-known bundle interface, which encompasses the app installed by the bundle (mysql), the application version (5.7.12), parameters and credentials required by the bundle, and the outputs generated by the bundle (connection string).


### Use Case Solutions

For each use case outlined above, how can we use these new match criteria?

* [Solution: Exclusive Resource](#solution-exclusive-resource)
* [Solution: Global Resource](#solution-global-resource)
* [Solution: Scoped Resource](#solution-scoped-resource)
* [Solution: User Provided Resource](#solution-user-provided-resource)
* [Solution: Polymorophic Resource](#solution-polymorphic-resource)

#### Solution: Exclusive Resource

Use a label that would not exist already so that a new installation is created just for this bundle in the current namespace.

Bundle
```yaml
dependencies:
  requires:
  - name: redis
    namespace: "{{ installation.namespace }}"
    labels:
      owner: "{{ installation.name }}"
```


#### Solution: Global Resource

Install OLM and do not set a namespace. The Porter Operator is installed into a namespace, and it reuses the global installation of OLM. When a match cannot be found, OLM is installed without a namespace set.

```yaml
dependencies:
  requires:
  - name: olm
```

* When more than one installation matches, installations in the current namespace are preferred over global installations.
* Note that this looks like today's dependency definition because global is the default right now.


#### Solution: Scoped Resource

The micro-services use a namespace criteria for their dependency on postgres. The first micro-service won't find a match, and installs postgres in the current namespace. The subsequent micro-services reuse that installation.

Share the dependency between application instances in the same namespace
```yaml
dependencies:
  requires:
  - name: postgres
    namespace: "{{ installation.namespace }}"
    labels:
      app: my-application
```

The labels let us have multiple applications (which can be comprised of micro-services) installed in a namespace and only share a postgres installation between micro-services within the same application.


#### Solution: User Provided Resource

SQL Server is either set up by a Porter bundle or manually because it's existing infrastructure. For existing infra, you define a bundle that just exposes the connection string via an output. The bundle itself doesn't modify any resources.

Bundle
```yaml
dependencies:
  requires:
  - name: sqlserver
    reference: azure/sqlserver # Use this bundle if we can't find an existing one
    version: v1.2.x

install:
- exec:
    description: "Install something that needs a connection string"
    commands: ./install.sh
    arguments:
    - "{{ bundle.dependencies.sqlserver.outputs.connection-string }}"
```

Meta Dependency Bundle used to share a resource provisioned outside of Porter. It doesn't do anything but return the connection string given to it when first "installed".
```yaml
name: shared-dev-sql-server

parameters:
- name: connection-string
  type: string
  path: /cnab/app/outputs/connection-string

outputs:
- name: connection-string
  type: string
  path: /cnab/app/outputs/connection-string
```

```console
$ porter install --reference myapp:v1.0.0 --dependency sqlserver=shared-dev-sql-server
```

* All dependencies support user-provided resources, without having to declare it.
* Porter does not apply the match criteria to the provided resource.
  If the bundle has the necessary parameters and outputs, it will be used.


#### Solution: Polymorphic Resource

Every bundle that installs MySQL can declare that it implements the well-known interface `io.cnab.interface.mysql` which indicates that a set of parameters and outputs are defined on the bundle.

```yaml
name: azure-mysql

dependencies:
  provides:
  - interface: "io.cnab.interface.mysql"
    version: 5.7.12

parameters:
- name: database-name
  type: string
  default: ""
- name: mysql-user
  type: string
  default: ""

outputs:
- name: connection-string
  type: string
```   

Any bundle that needs a MySQL database adds a dependency on the `io.cnab.interface.mysql` interface, specifies a MySQL version range, sets any parameters required by the mysql bundle, and lists the outputs it needs.

```yaml
dependencies:
  requires:
  - name: mysql
    interface:
      name: mysql
      version: 5.7.x
      outputs:
      - name: connection-string
        type: string
    parameters:
      database-name: wordpress
      mysql-user: wordpress
```

When Porter installs the bundle, it first evaluates existing installations for bundles that implement `io.cnab.interfaces.mysql` and have the required parameters and outputs.
This relies on a bit of the honor system as it is a well-known but not well-defined interface.
If you say that your bundles implements an interface and it turns out that it doesn't, then Porter be unable to use it as a dependency at runtime.


', Porter relies on two pieces of information to select a bundle to install MySQL: user-provided bundle reference and [Porter's bundle index](#porter-bundle-index).


### Porter Bundle Index

The Porter bundle registry is stored in the host environment, along side installation data and other stored data.
It is an index of bundles that Porter has pulled via such commands as `porter install`, or `porter explain`.
This index enables users to:

* See a list of bundles (not instalations) used by the team
* Search for bundles by name, label or interface
* Currate bundles that are recommended for an environment

In order to search for bundles that implement an interface, a search space needs to be defined.
In each environment, a different bundle is selected as a good implementation of an interface and registered with Porter.
For example, in a demo or local development environment a Helm provisioned on-cluster MySQL database is a simple default choice.
Whereas in production, the cloud provider's MySQL as a Service bundle would be more appropriate.


#### List Indexed Bundles

List and query bundles in the Porter bundle index.

```console
$ porter bundles list [--label KEY=VALUE] [--interface INTERFACE] [--name QUERY] [--keyword KEYWORD]
# List bundles in the index

$ porter bundles list --name mysql
# Bundles that have mysql in the name

$ porter bundles list --label app=mysql
# Bundles that have the label app=mysql

$ porter bundles list --interface io.cnab.interface.mysql
# Bundles that implement io.cnab.interface.mysql
# Display a * on the line of a bundle that is the default implementation

$ porter bundles list --keyword mysql
# Bundles that have the keyword mysql defined
```

#### Register a Bundle

Bundles are automatically registered in the index when they are pulled.
A bundle can be manually indexed with the `porter bundle add` command.
If a bundle was already set as the default, it is unset and the specified bundle becomes the default.

```console
$ porter bundle add --repository REPOSITORY [--interface INTERFACE]
# Pulls the bundle and indexes it

$ porter bundle add --repository ghcr.io/getporter/helm-mysql --interface io.cnab.interfaces.mysql
# Registers the helm-mysql bundle as the default implementation for io.cnab.interfaces.mysql
```

Note: I chose to use the verb "add" instead of register because it allows us to have consistent verbs across resources. Having a register alias may be useful as well.


#### Unregister a Bundle

A bundle can be removed from the bundle index with the `porter bundle remove` command.

```console
$ porter bundle remove --repository REPOSITORY
```

⛔️ Bundle index management, such as automatically removing unused bundles, scanning bundles for new versions, or removing all bundles, is out-of-scope of this proposal.


### Post-Install Dependency Management

At installation, it's clear that Porter should call the install action for any dependencies that do not already exist.
After that, each bundle may have different requirements for if the dependencies should be upgraded in lock-step, if the bundle is a decomposed set of services or has looser relationships.
Currently when a Porter action is executed on a bundle, that action is automatically executed on its dependencies as well.
In many of the use cases identified in this proposal, that is not the desired behavior.

New configuration at the bundle level and at the point when a bundle action is run should allow people to control this behavior.

All bundle actions (install, upgrade, invoke, uninstall) have a common flag, `--dependencies=true|false`, that let the user override the dependency behavior set in the bundle. This flag controlls whether or not Porter's dependency management step should be executed or not, and defaults to true.

The user can decide to install a bundle without its dependencies, manually passing in data that would have come from the dependency as credentials.

```console
$ porter install myapp --reference myorg/mybundle --dependencies=false --cred logger
# the logger credential contains the logger endpoint normally output by the logger dependency
```

They can upgrade just a single bundle, without upgrading its dependencies.

```console
$ porter upgrade myapp --dependencies=false
# Only upgrade the application and not the logger
```

When it comes time to uninstall, they can choose whether or not to remove the dependencies when no other installations are using the dependency.

```
$ porter uninstall myapp --dependencies=false
# Only uninstall myapp, do not remove unused managed dependencies
```

### Managed Dependency

When the bundle author defines the dependency, they can declare if the dependency should be managed together with the bundle as a unit (current behavior).
This is useful when an application is decomposed into multiple bundles, but it is intended to be managed together as a single unit, e.g. meta bundles.

The bundle depends upon its logger component which should be managed in lockstep with the bundle. The `lifecycle.managed` field indicates that all actions performed upon the bundle should also be applied to the dependency (defaulting `lifecycle.upgrade|invoke|uninstall` to true).

```yaml
dependencies:
  requires:
  - name: logger
    reference: myorg/mylogger
    version: v1.2.3
    lifecycle:
      managed: true
```

The Wordpress bundle depends upon a MySQL database which requires custom dependency management behavior. When upgrade or invoke is called on Wordpress, the MySQL dependency should also execute that action. However when Wordpress is uninstalled, MySQL should not be uninstalled by default.

```yaml
dependencies:
  requires:
  - name: logger
    reference: myorg/mylogger
    version: v1.2.3
    lifecycle:
      upgrade: true
      invoke: true
      uninstall: false
```


#### Detecting Unused Managed Dependencies

When a bundle is uninstalled, its managed dependencies may end up no longer being used.
For example, the bundle installed Wordpress and had an exclusive dependency on MySQL, so when Wordpress is uninstalled, MySQL could be classified as unused and optionally uninstalled as well. That example is unique because of data retention concerns.
Another example is a meta-bundle composed of stateless micro-services, where when the application is uninstalled dependencies like the logger are more clearly not required anymore and should be uninstalled as well.

### Unmanaged Dependency

Dependencies that are not declared as managed are not upgraded automatically when the current bundle is upgraded.
This is useful in cases where the dependency is shared with unrelated bundles such as a SQL Server installation.

### Dependency Resolution

🚧
* Resolve the entire graph, not just 1 level deep
* Identify shared dependencies in the graph based on match criteria. The same bundle dependency can either be shared between dependencies or some may get their own instance depending on the criteria.

___

## Implementation

<!--
After the PEP status is changed to implementable, when the PEP has been
implemented link to the pull request(s) here.
-->


---

## Backwards Compatibility

The dependencies section in porter.yaml is changing its format.
Currently the list of required bundles is directly underneath dependencies, and this proposal changes it to be under dependencies.requires.
Since we are pre 1.0, it's an acceptable breaking change but we will need to warn people before the release and call it out in the release notes.

___

## Security Implications

* Reusing outputs from existing installations

  This proposal allows you to reuse outputs from an existing installation.
  Porter does not have any ACLs on installation data, you either have access to the host environment or you don't.
  The way to isolate installation data from being accessed or reused by another installation is to use separate host environments.
  Users could do this previously through `porter installation outputs show OUTPUT --installation INSTALLATION` so it's not a new concern, just makes it easier.

___

## Rejected Ideas

* We considered the seemingly obvious solution of scanning a OCI registry for bundles that implement an interface, and rejected it because:
  * Use Case: The use case isn't to not have to think about which bundle is used for a dependency but being able to select from a variety of implementations.
  * Fit for Purpose: The implementation rarely doesn't matter and relies on the target environments infrastructure. Selecting an implementation and bundle author at random wouldn't make anyone happy.
  * Performance: Scanning registries would significantly slow down bundle installation.
  * Predictibility: A different bundle could be selected depending on the search order and the contents of the registry on any day.


---

## Open Questions

* I'm not sure yet how to handle passing credentials to the dependency that implements an interface, as that isn't possible to standardize in the interface.
  An azure mysql bundle will require different credentials than the google mysql bundle.
* Should the installation name label selector be defined in porter.yaml (i.e. a hard coded or parameterized dependency name) or should it only be a CLI flag?
  * I lean towards just a flag because we don't want people hard coding dependencies based on a dynamic value (installation name) or accidentally encourage people to make parameters to set dependency names when there are flags to do that.

* Are we okay with directly exposing the mechanisms for namespace or do we think there's value in making them more task based?
  I'm not sure that I have a use case for allowing namespace to be set to anything, we certainly don't want it hardcoded or again invite people to create parameters for it.
  For example instead of `namespace: "{{ installation.namespace }}"`, we could use `scope: namespace|global` which translates to that under the covers.
  By limiting the values in the porter.yaml, we would be making it easier to do it the right way.
* For the `dependencies.lifecycle` do we need both managed and the actions? I'm concerned it will confuse people since they are intended to be one or the other and not merged.
* Does user provided resources (i.e. creating a meta bundle for an existing resource created outside of porter) feel like _too_ much of a hack?
  It lets us use either external creds or outputs from a real bundle, and allow porter's resolution to "just work" without manually setting credentials everytime you need to use an external resource.
  Seems worth having since I know people asked for this regularly with service catalog.
* There may be a loose resemblance between the bundle index and helm chart repositories that we should investigate further.
