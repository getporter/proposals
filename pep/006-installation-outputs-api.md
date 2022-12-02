---
title: "Porter Outputs API"
number: "006"
status: "provisional"
owner: "@bdegeeter"
authors:
  - "@bdegeeter"
  - "@sgettys"
created: "2022-11-29"
type: "feature"
---

* [Abstract](#abstract)
* [Motivation](#motivation)
  * [Use Case: Installation outputs accessible via secret store](use-case-installation-outputs-accessible-via-secret-store)
  * [Use Case: InstallationOutputs CRD](#use-case-installationoutputs-crd)
* [Rationale](#rationale)
* [Specification](#specification)
* [Implementation](#implementation)
* [Backwards Compatibility](#backwards-compatibility)
* [Security Implications](#security-implications)
* [Rejected Ideas](#rejected-ideas)
* [Open Questions](#open-questions)

## Abstract

Support an internal API for access to installation outputs. This is the first phase of introducing an API, as such 
the scope is focused on minimal use cases with no external access support. In future iterations of this
work a public facing API shall be introduced.

___

## Motivation

https://github.com/getporter/operator/issues/63

When managing installations via the Porter Operator's Installation resource, there
is currently no native way to access outputs from the installation.

Now that we have working prototypes of services leveraging the Operator we've identified feature gaps.

Initial gaps were addressed by adding CRDs for Credential and Parameter sets. However,
it was not obvious/simple to get outputs.

Ideally the Operator would provide an InstallationOutputs resource
(or other programmatic access) that would be kept
in sync with an Installation.  This new resource would allow for controlled access to
outputs.


### Use Case: Installation outputs accessible via secret store

> I need installation outputs accessible via k8s secrets or external secret store (HashiCorp Vault/Azure KeyVault)

### Use Case: InstallationOutputs CRD

> I need Kubernetes native way to access outputs from an Operator driven installation

Populate an InstallationOutputs CRD to compliment an Installation resource. It should 
be kept in sync with the latest successful action from an Installation.

Populating an InstallationOutputs resource shall be optional and disabled by default.

## Rationale

One examples of a Operator based service is using a work around executing
an agent action of `porter installation -i <name> outputs list` along
with parsing of the job logs.

This is complicated, error prone, and not Kubernetes native.

## Specification

The API endpoints should map in a familar way to the Porter CLI

Looking at basic steps to view run outputs
```
porter installations list
porter installations runs list <installation-name>
porter installations outputs list <installation-name> #outputs of latest run
```

A rough protobuf service outline
```
service PorterBundle {
  //Returns a list of all installations
  rpc ListInstallations(google.protobuf.Empty) returns(ListInstallationsResponse) {}

  //Returns a list of all runs for a single installation
  rpc ListInstallationRuns(InstallationRequest) returns(ListInstallationRunsResponse) {}

  //Returns a list of all outputs for a single installation run
  //Must support a "latest" run id
  rpc ListInstallationRunOutputs(InstallationRunRequest) returns(ListInstallationRunOutputsResponse) {}
}

```



## Implementation

* Have the porter project deliver a API service binary in addition to CLI
  * This can then be managed by the Operator to reconcile an
    InstallationOutputs resource
* gRPC or ttRPC
  * This is initially an internal facing API
* Use https://buf.build

## Backwards Compatibility

This new feature should be compatible with 1.X and not support prior versions.

## Security Implications

* Who gets to access the outputs?
* What's the role of Kubernetes RBAC?
* How are sensitive outputs managed?

## Rejected Ideas


## Open Questions

* Can we leverage services which allow accessing external secret stores via k8s secrets?
  * i.e. write outputs to a k8s secret that is stored in Hashi Vault
* What's public facing? Just a new InstallationOutputs resource?

