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
  * [Use Case: Exclusive Resource](#use-case-exclusive-resource)
* [Rationale](#rationale)
* [Specification](#specification)
* [Implementation](#implementation)
* [Backwards Compatibility](#backwards-compatibility)
* [Security Implications](#security-implications)
* [Rejected Ideas](#rejected-ideas)
* [Open Questions](#open-questions)

## Abstract

Support an initial API for access to installation outputs.

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


### Use Case: Populate an InstallationOutputs CRD to compliment an Installation resource.

> I need Kubernetes native way to access outputs from an Operator driven installation


## Rationale

One examples of a Operator based service is using a work around executing
an agent action of `porter installation -i <name> outputs list` along
with parsing of the job logs.

This is complicated, error prone, and not Kubernetes native.

## Specification

TBD

## Implementation

* Have the porter project deliver a API service binary in addition to CLI
  * This can then be managed by the Operator to reconcile an
    InstallationOutputs resource
* gRPC, ttRPC, GraphQL, REST: which to use where and why
* What's public facing?  Just a new InstallationOutputs resource?

## Backwards Compatibility

This new feature should be compatible with 1.X and not support prior versions.

## Security Implications

* Who gets to access the outputs?
* What's the role of Kubernetes RBAC?
* How are sensitive outputs managed?

## Rejected Ideas


## Open Questions

