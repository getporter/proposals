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
  * [Use Case: Application Resource](#use-case-application-resource)
  * [Use Case: User-Provided Resource](#use-case-user-provided-resource)
  * [Use Case: Bundle Interface](#use-case-bundle-interface)
* [Rationale](#rationale)
* [Specification](#specification)
  * [porter.yaml](#porter-yaml)
  * [Bundle Interfaces](#bundle-interfaces)
  * [Wiring Dependencies](#wiring-dependencies)
  * [Dependency Resolution](#dependency-resolution)
  * [Use Case Solutions](#use-case-solutions)
    * [Solution: Exclusive Resource](#solution-exclusive-resource)
    * [Solution: Global Resource](#solution-global-resource)
    * [Solution: Application Resource](#solution-application-resource)
    * [Solution: User-Provided Resource](#solution-user-provided-resource)
    * [Solution: Bundle Interface](#solution-bundle-interface)
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

My custom Kubernetes operator depends up on having Flux installed on the cluster.
If is already present, reuse it.


### Use Case: Application Resource

> My micro-services share a common dependency

My application has 4 micro-services and a shared postgres database.
When I install the first service, a database should be created for it.
When I install the other services, I want to share the database (and the connection string) between them.


### Use Case: User-Provided Resource

> I already installed the dependency previously

I don't want Porter to install SQL Server for me, but I do want Porter to help connect the outputs from a named installation into another bundle.
For example, a manually set up on-premise shared database server, the 64 core beast that you keep in the office closet.
I want to automatically inject the connection string for the database into the bundles I install.

### Use Case: Bundle Interface

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

The heart of the solution for all of the above use cases is reducing them to a single decision: "GetOrCreate".
The bundle declares a set of criteria that is used to identify an existing installation that satisfies the dependency.
If one cannot be resolved, then a new installation is created.

### porter.yaml

Below is the proposed dependencies section format:

```yaml
dependencies:
  # unordered list of bundles that this bundle depends upon
  requires:
    # A name or alias for the dependency throughout porter.yaml
    - name: DEPENDENCY_NAME
      # Criteria for using a bundle for the dependency
      bundle:
        # A full bundle reference that can be used as the default implementation of the bundle
        reference: FULL_BUNDLE_REFERENCE
        # Criteria for the bundle version used
        # Use vX.Y.Z-0 to allow prerelease versions to be resolved by Porter
        version: SEMVER_RANGE # See https://github.com/Masterminds/semver (v3 format)
        # An interface defining how the dependency will be used by this bundle
        # Porter always infers a base interface based on how the dependency is used in porter.yaml
        interface:
          # Specifies a bundle to use as the interface on top of how the bundle is used.
          reference: FULL_BUNDLE_REFERENCE
          # Specifies additional constraints that should be added to the bundle interface. Either bundle or reference may be specified but not both.
          # By default porter only requires the name and the type to match, additional jsonschema values can be specified to restrict matching bundles even further.
          bundle:
            parameters:
              - name: PARAMETER_NAME
                type: PARAMETER_TYPE
            credentials:
              - name: CREDENTIAL_NAME
                type: CREDENTIAL_TYPE
            outputs:
              - name: OUTPUT_NAME
                # When an $id is specified, the dependency must use the same $id value for the parameter/credential/output and the name does not have to match the name declared on the interface. Allows a bundle to implement an interface with different naming conventions.
                $id: URI
      # Configuration for the installation that represents the dependency
      installation:
        # When porter creates a new installation for this dependency, apply the following labels
        labels:
          LABEL_KEY: LABEL_VALUE
        # Criteria for selecting an existing installation to use for the dependency
        criteria:
          # Only match the bundle interface, and not the bundle repository from the reference. Defaults to false, so that the bundle must be the same.
          matchInterface: BOOL
          # Require that the existing installation must be in the same namespace and cannot be a global installation. Defaults to false, which allows reusing global installations as dependencies.
          matchNamespace: BOOL
          # Allow reusing an existing installation that does not have the labels specified above. By default the labels must match to reuse an installation.
          ignoreLabels: BOOL
```


### Bundle Interfaces

Bundle interfaces are relevant when reusing installations to satisfy a dependency and when using a different bundle that the default implementation provided by the dependency.

The interface is used to validate that a dependency can be used in the following situations:
* When a version range is specified and Porter attempts to use the highest allowed version.
* When the user overrides the bundle used for a dependency.
* When Porter reuses an existing installation for a dependency, though only the interface outputs are compared.

Porter will always enforce a base bundle interface defined by how the dependency is used.
If the parent bundle expects to pass a parameter or credential, the dependency must have a parameter or credential upon which it can be mapped.
Similarly for outputs, if the parent bundle uses an output from the dependency, anything used for that dependency must generate that output.

The bundle author can influence the interface of a dependency in by providing a bundle.json document (either as a reference or embedded in the dependency) that provides additional jsonschema for the parameters, credentials, and outputs.

It's unrealistic to expect that every bundle will use the same consistent names for its parameters and outputs as other bundles that represent the same type of resource.
We can't expect all bundles that install MySQL for example to use connstr for its output connection string for example.
To accommodate this, bundle authors can agree on using a well-known identifier to indicate what that parameter, credential or output represents.

When a bundle applies the identifier, it allows a consuming bundle that would depend upon it to rely on the identifer instead of requiring the name to match as well.

```yaml
credentials:
  - name: admin-kubeconfig
    $id: "porter.sh/interfaces/kubernetes.config.cluster-admin"
```

When wiring up the bundle, Porter uses the identifier to map from the name item in the dependency to how it is used in the consuming bundle.
This allows a bundle to refer to the output connection string as "connstr", while some bundles defined the output as "connection-string" and others as "dbConn", etc.

Porter can help maintain a set of well-known identifiers and explain what it represents so that other authors can use them.
Bundle authors who are interested in having the widest base possible of users will be motivated to apply the identifier so that their bundle can be used across platforms.

### Wiring Dependencies

Dependencies defined in a bundle can be "wired" up to the parent bundle or other dependencies parameters, credentials, and outputs.
Following the CNAB Dependencies spec, the dependencies of a bundle do not "leak" into the parent bundle's interface.
So if a bundle depends upon another bundle that requires a credential, it is the responsibility of the parent bundle to set the dependency's credential, possibly by defining a credential on the parent bundle and then passing the credential to the dependency.
When a user interacts with a bundle, they cannot directly set parameters and credentials of the parent bundle's dependencies, and must aways go through the parent bundle only.

Below are some examples of how this wiring can look:

The parent bundle's kubeconfig credential is passed to the redis dependency's kubeconfig credential.

```yaml
credentials:
- name: kubeconfig

dependencies:
  requires:
    - name: redis
      bundle:
        reference: getporter/redis:v1.0.0
      credentials:
        kubeconfig: bundle.credentials.kubeconfig # Note that this isn't a template variable though it uses the same syntax
```

The mysql dependency's connection-string output is passed to the myapp dependency's connstr parameter.
This indicates to Porter that the mysql dependency should be executed before the myapp dependency.

```yaml
dependencies:
  requires:
    - name: myapp
      bundle:
        reference: getporter/myapp:v1.0.0
      parameters:
        connstr: bundle.dependencies.mysql.outputs.connection-string
    - name: mysql
      bundle:
        reference: getporter/mysql:v5.7.13
```

* An output of a dependency can be passed to a credential of another dependency.
* An output of a dependency be used as the output value of the parent bundle's output.

### Dependency Resolution

Below is the precedence in which a dependency is resolved to a bundle or existing installation:

1. Bundle references or existing installations provided by the end-user.
1. Existing installations that match the installation criteria.
  * Matching the bundle reference is preferred over only matching the interface, even when matchInterface is true.
  * Installations in the installation namespace are preferred over global installations when matchNamespace is false).
  * Installations with matching labels are preferred over installations where the labels do not match when ignoreLabels is true.
1. The highest version of the dependency's reference bundle that matches the defined version constraints.
1. The reference specified on the dependency (the default implementation).

In the future, Porter may keep a local list of bundles from which implementations may be resolved, but that is out of scope for this proposal.

After Porter resolves every dependency in the graph, it generates an execution plan that specifies the order that the bundles must be executed.
When an output from a dependency is set as a source to another dependency parameter or credential, that creates an additional constraint on the dependency graph.
The resulting execution plan should include not only the order of the bundles but how the parameters, credentials and outputs are wired.
If an execution plan can be generated, then the entire bundle graph should be executable without failing mid-run because the source of a parameter or credential cannot be determined, or relies up on a bundle that has not been executed yet.

### Use Case Solutions

For each use case outlined above, how can we use solve the original use cases?

* [Solution: Exclusive Resource](#solution-exclusive-resource)
* [Solution: Global Resource](#solution-global-resource)
* [Solution: Scoped Resource](#solution-scoped-resource)
* [Solution: User-Provided Resource](#solution-user-provided-resource)
* [Solution: Bundle Interface](#solution-bundle-interface)

#### Solution: Exclusive Resource

Example: 
> an application requires a dedicated redis cache per installation

The dependency below will create a new installation of the getporter/redis:v1.0.2 bundle unless there is an existing installation of that bundle in the installation namespace that is labeled with the name of the current installation.

In practice, this will always result in a new installation for the dependency unless the bundle was uninstalled, leaving the dependency alone, and then it was reinstalled with the same name, i.e. re-installing can pick up the same database it had last time.

```yaml
dependencies:
  requires:
    - name: redis
      bundle:
        reference: getporter/redis:v1.0.2
      installation: 
        labels:
          installation: "{{ installation.name }}"
```


#### Solution: Global Resource

Example:
> My custom Kubernetes operator depends up on having Flux installed on the cluster.
> If is already present, reuse it.

The dependency below will reuse an installation of getporter/flux:v2.x in the installation namespace first, and if not present, try to reuse a global installation.
Otherwise, it will install getporter/flux:v2.1.3 in the installation namespace.
If the end-user wants Flux to be installed globally, they must install it into the global (empty) namespace first.

```yaml
dependencies:
  requires:
  - name: flux
    bundle:
      reference: getporter/flux:v2.1.3
      version: 2.x
```

#### Solution: Application Resource

Example:
> My application has 4 micro-services and a shared postgres database.
> When I install the first service, a database should be created for it.
> When I install the other services, I want to share the database (and the connection string) between them.

The dependency below will reuse an installation of getporter/postgres:v2.x in the installation namespace only if it has the app=my-application label.
Otherwise, it will install getporter/postgres:v2.3.4 in the installation namespace.

```yaml
dependencies:
  requires:
  - name: postgres
    bundle:
      reference: getporter/postgres:v2.3.4
      version: 2.x
    installation:
      labels:
        app: my-application
      criteria:
        matchNamespce: true
```

#### Solution: User-Provided Resource

Example:
> I want Porter to automatically reuse existing infrastructure that I deployed outside of Porter.

For existing infra, define a bundle that just exposes the connection string via an output.
The bundle itself doesn't manage any resources.
When the bundle is installed, Porter remembers the provided connection string and can provide it to any installation that uses it as a dependency.

```yaml
name: shared-dev-sql-server

parameters:
- name: connection-string
  type: string
  path: /cnab/app/outputs/connection-string

outputs:
- name: connection-string
  type: string
  $id: "azure-sql-server-connection-string"
  path: /cnab/app/outputs/connection-string
```

The dependency below will reuse any installation that defines an output with the id "azure-sql-server-connection-string", regardless of the installation's bundle.
It maps the output found on the installation to the name "dbCon", allowing even bundles with different names for the output to be used.

```yaml
dependencies:
  requires:
  - name: sqlserver
    bundle:
      reference: azure/sqlserver:v1.2.68
      interface:
        outputs:
          - name: dbCon
            $id: "azure-sql-server-connection-string"
    installation:
      matchInterface: true

install:
- exec:
    description: "Install something that needs a connection string"
    commands: ./install.sh
    arguments:
    - "{{ bundle.dependencies.sqlserver.outputs.dbCon }}"
```

Here is what it would look like for an administrator to register the existing server and then use it.

```console
$ porter install --reference shared-dev-sql-server:v0.1.0 -p connection-string=existing-connection-string
# registers the existing server with Porter

$ porter install --reference myapp:v1.0.0
# Porter automatically reuses the existing server to satisfy the dependency
```

#### Solution: Bundle Interface

Example:
> I want my bundle to be able depend on a bundle that provisions a specific version of MySQL and outputs the connection string, without having to take a hard dependency on a specific implementation.

The dependency below 
```yaml
dependencies:
  requires:
  - name: mysql
    bundle:
      interface:
        outputs:
          - name: dbCon
            $id: "mysql-5.7-connection-string"
    installation:
      matchInterface: true

install:
- exec:
    description: "Install something that needs a connection string"
    commands: ./install.sh
    arguments:
    - "{{ bundle.dependencies.sqlserver.outputs.dbCon }}"
```

When Porter installs the bundle, it first tries to reuse an existing installation that has an output with id "mysql-5.7-connection-string".
If an existing installation cannot be found, the bundle cannot be installed and the user is prompted to specify which bundle to use.
This allows an author to require a resource, without providing a default implementation, which can be difficult when writing a bundle that can be used in multiple platforms.

When matching existing installations against the bundle interface, only outputs are considered, even if the interface defines parameters and credentials as well.

## Implementation

<!--
After the PEP status is changed to implementable, when the PEP has been
implemented link to the pull request(s) here.
-->


---

## Backwards Compatibility

The changes proposed here are not backwards compatible with the existing dependencies feature in v0.38.
When this feature is released, the changes to porter.yaml will require a major version increase.
We are not targeting this change for the v1.0.0 milestone.
In addition, when we save the dependencies metadata to bundle.json, it will need to use a new custom extension name, such as `io.cnab.dependencies@v2`.
The exact extension name is TBD and will be decided in the CNAB spec.

___

## Security Implications

* Reusing outputs from existing installations

  This proposal allows you to reuse outputs from an existing installation.
  Porter does not have any ACLs on installation data, you either have access to the host environment or you don't.
  The way to isolate installation data from being accessed or reused by another installation is to use separate host environments.
  Users could do this previously through `porter installation outputs show OUTPUT --installation INSTALLATION` so it's not a new concern, just makes it easier.

___

## Rejected Ideas

N/A

---

## Open Questions

### How can we use an interface when implementations have different credentials?

A. It is very likely that for bundles that implement a resource on different platforms that the credentials will not be the same between possible bundle implementations.
The google bundle will require different credentials than the azure bundle.
This isn't true for all resources, apps that deploy to kubernetes for example will only need a kubeconfig, regardless of the cluster vendor.

In this case, the user can install the dependency independently, installing the platform specific bundle first providing the credentials it requires, and then Porter will reuse it when the application bundle is subsequently installed.


### Do we still want to use an array for the dependencies?

Q. The current format uses an array to create an explicit ordering of the dependencies.
Now that Porter resolves its own ordering, should we stick with the array to be consistent with how we define other things, like parameters, or should we use a map to reinforce that the ordering is not based on the order of the dependencies in the array?

A. For consistency and better autocomplete experience, we will stick with an array.
