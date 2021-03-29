---
title: "Labels and Namespaces"
number: "004"
status: "provisional"
owner: "@carolynvs"
authors:
  - "@carolynvs"
created: "2021-03-22"
type: "feature"
---

## Abstract

We need a way to organize and isolate installations in an environment with multiple people managing installations.


## Motivation

When multiple people are working with the same installations and Porter data, it becomes easy to "collide" and step on each other's toes.
For example, two people want to install the same application and the first person gets the preferred installation name.
Or when there is a team of people running bundles in an environment, it is hard to keep track of which is used for what and who owns what.


## Rationale

This is already a solved problem in Kubernetes, where they use labels and namespaces to organize and isolate data.


## Specification

### Labels

Labels are used to organize Porter's documents.
Labels can be used to indicate lightweight ownership, environment, and purpose.

```
owner=carolynvs
cnab.io/app=myApp
cnab.io/appVersion=v1.2.3
env=dev
```

Labels are stored as string key/value pairs, using the same schema as [Kubernetes labels].
The requirements below are taken from the Kubernetes label documentation, with the reserved prefix updated to be specific to Porter.

Labels are key/value pairs. Valid label keys have two segments: an optional prefix and name, separated by a slash (/).
The name segment is required and must be 63 characters or less, beginning and ending with an alphanumeric character ([a-z0-9A-Z]) with dashes (-), underscores (_), dots (.), and alphanumerics between.
The prefix is optional. If specified, the prefix must be a DNS subdomain: a series of DNS labels separated by dots (.), not longer than 253 characters in total, followed by a slash (/).

If the prefix is omitted, the label Key is presumed to be private to the user.
Automated system components (e.g. Porter Operator) which add labels to end-user objects must specify a prefix.

The porter.sh/ prefix is reserved for Porter's use.

Valid label values:

* must be 63 characters or less (cannot be empty),
* must begin and end with an alphanumeric character ([a-z0-9A-Z]),
* could contain dashes (-), underscores (_), dots (.), and alphanumerics between.

The following documents support labels:

* [Bundles](#bundle-labels)
* [Installations](#installation-labels)
* [Claims](#claim-labels)
* [Results](#result-labels)
* [Outputs](#output-labels)
* [Credential and Parameter Sets](#credential-and-parameter-set-labels)

I have proposed supporting labels officially in 
[CNAB Installation State Specification], which would cover both storing the labels and govern their format.

[Kubernetes labels]: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set

#### Bundle Labels

A new section in porter.yaml stores labels:

```yaml
name: mybun
version: v0.1.0

labels:
  cnab.io/app: myapp
  cnab.io/appVersion: v1.2.3
  internal: "true"
```

The [CNAB Installation State Specification] proposes as a first class value on the bundle, reserving the `cnab.io/` prefix, and defining common labels that tools can use to filter bundles:

* cnab.io/app - The name of the application installed by the bundle.
* cnab.io/appVersion - The version of the application installed by the bundle.

If accepted labels would be stored in the bundle.json like so:

```json
{
  "name": "mybun",
  "version": "v0.1.0",
  "labels": {
    "cnab.io/app": "myapp",
    "cnab.io/appVersion": "v1.2.3",
    "internal": "true"
  }
}
```

#### Installation Labels

When a bundle is executed, the labels defined on the bundle are copied and used as labels on the installation.
Users can apply additional labels using the `--label-installation` flag, which can be repeated, and uses the following format: KEY=VALUE.

Let's consider a bundle that deploys v1.2.3 of myApp.
The bundle has two labels defined, `cnab.io/app=myApp` and `cnab.io/appVersion=v1.2.3`, and when the bundle is installed, those labels are applied to the installation.
Later when the installation is upgraded to a newer version of the bundle that deploys myApp v1.2.4, the labels on the bundle are again applied to the installation, updating the `cnab.io/appVersion` label to `v1.2.4`.

Labels can be applied to an installation when the bundle is installed:
```
porter install wordpress --reference getporter/wordpress:v1.0 --label-installation owner=carolynvs
```

Labels can be applied after an installation is created using the `porter installation label` command:
```
$ porter installation label set wordpress --label env=dev --label internal=true
# add (or updates) env=dev and internal=true to the wordpress installation

$ porter installation label delete wordpress --label env
# Remove the env label from the wordpress installation
```

🚨 The CNAB Spec does not yet cover storing installation data, so I have proposed adding a [CNAB Installation State Specification] that can store additional metadata associated with the installation.

```json
{
  "name": "wordpress",
  "created": "TIMESTAMP",
  "bundleRepository": "getporter/wordpress",
  "version": "0.1.0",
  "labels": {
    "owner": "carolynvs",
    "cnab.io/app": "wordpress",
    "cnab.io/appVersion": "v1.2.3"
  },
  "custom": {}
}
```

Users can filter by labels when listing installations:

```console
$ porter installations list --label owner=carolyn
# Filter installations by the label owner=carolyn
```

#### Claim Labels

In Porter a claim is represented as a "run".
Each time porter install/upgrade/invoke/uninstall is executed, that generates a claim (except when the action is stateless such as io.cnab.status).
Labels can be applied to the claim with the `--label` flag.
For example, the Porter Operator could set `--label cnab.io/executor=porter-operator` when it executes a bundle, allowing us to see in the history if an action was executed manually or via the operator.

```
porter install wordpress --reference getporter/wordpress:v1.0 --label porter.sh/client=porter-operator --label porter.sh/clientVersion v0.3.0
```

**claim**
```json
{
  "id": "CLAIM_ID",
  "installation": "wordpress",
  "action": "install",
  "labels": {
    "cnab.io/executor": "porter",
    "cnab.io/executorVersion": "v0.36.0",
    "porter.sh/client": "porter-operator",
    "porter.sh/clientVersion": "v0.3.0",
    "porter.sh/porterRuntimeVersion": "v0.33.0"
  }
}
```

Porter should always set the `cnab.io/executor` label to "porter" and `cnab.io/executorVersion` label to the current version of the Porter client, using `porter.sh/porterRuntimeVersion` to record the version of Porter that was embedded in the bundle (if available).
When the Porter Operator executes an installation, it should set `porter.sh/client` to "porter-operator", and `porter.sh/clientVersion` to the version of the Porter Operator. By default when these labels are not already provided, the Porter CLI should set `porter.sh/client` and `porter.sh/clientVersion` to the same values it uses for the executor labels.

Knowing which version of Porter was used to execute each part of the bundle (client/runtime) will help us debug and troubleshoot problems with bundles in production.

#### Result Labels

The spec allows for labels to be set on a result however we will not expose that in Porter.
It is essentially reserved for custom tools and future use in Porter.

#### Output Labels

The [CNAB Installation State Specification] adds labels to the output definition in a bundle.
When an output is generated, labels defined on the output are added to the output metadata on the claim result.

**bundle.json**
```
"outputs":{
  "clientCert":{
    "definition":"x509Certificate",
    "path":"/cnab/app/outputs/clientCert",
    "labels": {
      "format": "PEM"
    }
  },
```

**result**
```json
{
  "id": "RESULT_ID",
  "outputs": {
    "clientCert": {
      "contentDigest": "abc123",
      "generatedByBundle": "true",
      "labels": {
        "format": "PEM"
      }
    }
  }
}
```

#### Credential and Parameter Set Labels

Credentials and Parameter sets both support defining labels.
Below is the representation of labels in a credential/parameter set.

```json
{
  "name": "azure-dev",
  "created": "TIMESTAMP",
  "modified": "TIMESTAMP",
  "labels": {
    "env": "dev"
  }
}
```

Labels can be specified when they are created:

```console
$ porter credentials generate NAME --reference REFERENCE --label owner=carolyn

$ porter parameters generate NAME --reference REFERENCE --label env=dev
```

Labels can be applied afterward with the edit command or with the new label sub-resource:

```console
$ porter credential label set NAME --label rotate-on=2022-03-01

$ porter parameters label delete NAME --label test-only
```

They can also be filtered by label when listing:

```console
$ porter credentials list --label env=prod

$ porter parameters list --label env=prod
```

### Namespaces

Namespaces allow us to isolate data within Porter's backend storage.
For example a developer could create a namespace with their username and then filter all their commands by this namespace so that they only see their own data, and not the rest of the team's.

The following documents support namespaces:

* [Installations](#installation-namespace)
* [Claims](#claim-namespace)
* [Results](#result-namespace)
* [Credential and Parameter Sets](#credential-and-parameter-set-namespace)

I have proposed supporting namespaces officially in 
[CNAB Installation State Specification], which would cover both storing the namespace and govern its format.

#### Installation Namespace

An installation can be defined within a namespace or globally. Global installations have the namespace set to an empty string ("").
The --namespace flag has a short flag of -n and defaults to an empty string. All installation commands should support --namespace.

```console
$ porter install INSTALLATION --reference REFERENCE [--namespace|-n NAMESPACE]
# install in the specified namespace, or globally if omitted

$ porter installation show INSTALLATION [--namespace|-n NAMESPACE]
# show the specified installation
```

An installation's namespace cannot be changed once set.

**installation**

```json
{
  "name": "wordpress",
  "created": "TIMESTAMP",
  "bundleRepository": "getporter/wordpress",
  "version": "0.1.0",
  "namespace": "carolyn"
}
```

The list command always applies the --namespace flag, even when not set.
When --namespaces is set to `*`, installations in all namespaces are returned.

```console
$ porter list
# displays only global installations

$ porter list --namespace carolyn
# displays installations in the carolyn namespace

$ porter list --namespace *
# displays all installations across namespaces
```

#### Claim Namespace

Claims store a namespace on its document to assist with querying, however claims are _always_ in the same namespace as the installation.
The claim namespace is not exposed to the end-user other than when they request documents that include a claim, such as when the --output json flag is used.

**claim**
```json
{
  "id": "CLAIM_ID",
  "installation": "wordpress",
  "action": "upgrade",
  "namespace": "carolyn"
}
```

#### Result Namespace

Results store a namespace on its document to assist with querying, however results are _always_ in the same namespace as the installation.
The result namespace is not exposed to the end-user other than when they request documents that include a result, such as when the --output json flag is used.

**claim**
```json
{
  "id": "RESULT_ID",
  "namespace": "carolyn"
}
```

#### Credential and Parameter Set Namespace

Credential and parameter sets can be defined within a namespace or globally. Global credential and parameter sets have the namespace set to an empty string ("").
The --namespace flag has a short flag of -n and defaults to an empty string and should be defined on all commands so that we can either filter by the namespace or uniquely identify a document using its name+namespace.

```console
$ porter credentials list
# Displays global credential sets

$ porter parameters list --namespace carolyn
# Displays parameters in the carolyn namespace

$ porter credentials list --namespace *
# Displays all credentials across namespaces

$ porter credentials show NAME [--namespace|-n NAMESPACE]
# Show the credential set matching the name and namespace

$ porter parameters delete NAME [--namespace|-n NAMESPACE]
# Delete the specified parameter.
```

## Implementation

<!--
After the PEP status is changed to implementable, when the PEP has been
implemented link to the pull request(s) here.
-->


## Backwards Compatibility

The namespace for existing documents is defaulted to the global namespace, an empty string.
So users with existing data will still see the same results when they run commands like list, or show.
A storage migration will be required to initialize the data however.

```
$ porter storage migrate
# Add empty namespaces to existing documents and create documents to represent installations using existing claim data.
```

Users with older clients won't be able to query data saved by a version of Porter that supports labels and namespaces
because they will be stored with a higher storage schema.

## Security Implications

None


## Rejected Ideas

None


## Open Questions

None

[CNAB Installation State Specification]: https://github.com/cnabio/cnab-spec/pull/411
