---
title: "Mixins are Bundles"
number: "005"
status: "provisional"
owner: "@carolynvs"
authors:
  - "@carolynvs"
created: "2022-03-03"
type: "feature"
---

## Abstract

Let's take advantage of the distribution and security features of bundles for distributing and executing mixins.

## Motivation

Initial Proposal: https://github.com/getporter/porter/discussions/1499

Currently mixins are binaries that are distributed by the porter client by downloading the mixin specified by the user and saving it to the porter home directory (PORTER_HOME).
When a bundle is built, the mixin binaries located in PORTER_HOME are copied into the bundle's invocation image.
This is a proposal to distribute the mixin binaries in a bundle instead of distributing the binaries directly.

### Security

___

#### Verify Provenance

> I want to be able to verify that the mixins in a bundle came from their stated source.

Right now, we communicate the name of the mixins included in a bundle, and it is up to the user to decide if the bundle was built securely and actually contains the correct version of the stated mixins.
A bundle author could use an arbitrary binary named after the mixin, and the end-user couldn't tell the difference.
End-users should be able to inspect a bundle, verifying the mixins source and signatures, before running it.

#### Secure Distribution

> I want to distribute mixins using established, secure cloud infrastructure.

By switching from manual binary hosting and distribution to bundles, OCI registries, and Notary, we can take advantage of Docker's security features, along with the features from the CNAB security specification.

#### Isolate Sensitive Data
> I want to limit which mixins can access bundle credentials. For example, only the az mixin should have access to my Azure credentials, not the exec mixin.

In addition to distribution, there is a security concern around limiting access to sensitive data in a bundle.
Every mixin in a bundle has access to every parameter, credential, and output.
You may trust your Azure credentials with the az CLI, but exposing it to unrelated mixins is an unnecessary risk.
Executing mixins inside a bundle allows us to limit the exposure of sensitive data on a need to know basis.
For example, the az mixin would have access to your Azure credentials, but the exec mixin that can run arbitrary shell scripts would not.

### Developer Experience
___

#### Reproducible Builds
> I want my bundle to declare not only the mixin used, but its version and source.

Bundle authors are responsible for tracking which version they use with each bundle and ensuring that version is installed before building the bundle, which is a major gap in our mixin functionality.
Before a bundle can be built, it is the author must install the mixins used by the bundle, and guess at the proper version since the mixin version is not stored in the bundle's metadata.
There should be enough information in the bundle definition such that Porter can download and use the mixins automatically.
This would enable reproducible builds of bundles, because now all the information required to build a bundle is included in the porter.yaml, such that a CI server can replicate what the author did locally.

#### Remove Artificial Restrictions

___

> I want to write mixins that are based on different OS distributions.

Currently, mixin and bundle authors are restricted to using Debian distributions that include apt because mixins are installed into the bundle's invocation image.
Separating mixins from bundles allows both mixin and bundle authors to define their invocation image however they require.
For example, a company can base their invocation images on their secured base image of RHEL that they keep patched and use as a common base image for all their images.

### Performance
___

#### Speed up Build, Publish, and Pull
> When I install two bundles that use the same mixins, the second bundle should pull faster.

Mixin content is static, but because it is copied into the invocation image, the content cannot be cached and shared between invocation images.
This creates large invocation images cannot reuse common layers data between bundles.
Due to the large invocation images, pushes and pulls are slow which impact nearly every porter command.
It is also a significant disk space cost that is disruptive to local development because you are easily chewing up hundreds of gigabytes due to inefficient use of docker caching.

#### Reduce Bundle Size
> I want smaller bundles.

With mixins distributed as bundles, the invocation image only contains files unique to the bundle.
The mixin is not embedded in the bundle, and so will be cached by the docker daemon.
This will reduce bundle sizes by an order of magnitude, for example from 1.2GB to less than 50MB, because it does not include a full linux distribution, developer tools, and dependencies.

### Impact

When this proposal is implemented:

* Mixin authors no longer compile their mixin to support multiple OS/ARCH combinations, and instead they use Porter to package their mixin into a bundle.
* Mixin authors no longer distribute binaries directly and instead publish their mixin to OCI registries with Porter.
* Bundle authors define not only the mixin name, but also its source and version. They no longer are responsible for micromanaging swapping which versions of mixins are installed on their local machine and CI server.
* End-users will now be able to see the exact version and source of the mixins used by a bundle, and verify their signature and provenance.

## Rationale

A lot of this design is driven by how mature tools in the ecosystem solve secure distribution, such as Docker's layer caching, image distribution, and Terraform's package locking scheme.
Reusing existing security solutions that are actively maintained is a much more solid foundation than implementing this ourselves.

This feature was originally suggested at https://github.com/getporter/porter/issues/975 and by @sestegra in https://github.com/getporter/porter/discussions/1311.

## Specification

TBD after this proposal is provisional.

<!--
The technical specification should describe the command and/or configuration
syntax and semantics of any new feature.

* If this is a command, we are looking for what the `porter help` would look
  like: description of command, arguments, flags, default behavior and error
  handling.

* If this is a syntax change to a configuration file, define the allowed syntax,
  at least one example per use case, covering defaults and error handling.

* All PEPs will be reviewed for user experience. So make sure to think about the
  common use case, how people can accomplish more advanced scenarios, precedence
  from existing Porter features or other tools in the ecosystem, and how the
  change fits into Porter workflows and tasks.

The spec should be detailed enough that someone other than the PEP
authors can understand what needs to be implemented.
-->


## Implementation

Links to pull requests implementing this solution will be added after the proposal is marked implementable.


## Backwards Compatibility

This includes breaking changes that require a major version bump in Porter.
Bundles built with older versions of Porter are not impacted by this change.
Bundles built with this new feature are not compatible with older versions of Porter.

## Security Implications

None identified.


## Rejected Ideas

None identified.


## Open Questions

### Should plugins be distributed as bundles too?
This would require additional design for how to communicate with the gRPC service inside the bundle.
Let's look into it but keep it out of scope for this proposal.
