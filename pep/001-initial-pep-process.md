---
title: "Initial Porter Enhancement Proposal Process"
number: "001"
status: "proposed"
owner: "@carolynvs"
authors:
  - "@carolynvs"
created: "2020-10-29"
type: "meta"
---

## Abstract

Let's define a process to propose and evaluate changes to the Porter project.
The Porter Enhancement Proposal (PEP) process is based on the [Python
Enhancement Proposal] with some parts also borrowed from Helm and Kubernetes. A
PEP is a document that explains why a change is desirable and proposes how to
implement it.


## Motivation

Adopting a proposal process will improve how proposals are handled and increase
transparency with the community. Right now, people make proposals either in an
issue or on our forum or slack messages, etc, we hash it out there, sometimes
making a new issue to track the final set of work, and then start on it.

The hodge-podge nature of our informal process makes it difficult to engage in
these discussions, and give people a reasonable turnabout time on feedback. It
takes a while to read through them, understand the use cases, potential impact
of the change, understand if there are gaps or unintended consequences, etc and
that usually ends up looking like no response from maintainers for too long. Not
to mention that it's difficult to track and keep on top of suggestions when they
are scattered everywhere. A formal process for proposals would set expectations
for both contributors and maintainers, allowing maintainers to quickly weigh in
on “Is this worth pursuing” without implicitly giving people the false
impression that they have performed all of the above due diligence and that it’s
approved or ready to start on.

By defining a process and identifying what due diligence needs to happen before
a proposal can be considered and approved, it would help us make changes more
safely, and collaborate on a design with the interested community members.


## Rationale

Three proposal processes were considered and used to create the proposed Porter
Enhancement Proposal:

* [Python Enhancement Proposal] - This is the foundation of the other two
  proposal processes. It's been around for a very long time and provides a great
  template for us to start with. Some changes are necessary due to the
  differences in Porter vs Python. Porter is an implementation, whereas Python
  is a specification. That is why the Kubernetes and Helm processes were
  incorporated as well.

* [Kubernetes Enhancement Proposal] - The PEP statuses were based on the KEP
  process since it is a better fit to say a PEP is "implementable" and
  "implemented" vs. having people write a reference implemenation of a spec
  (which only makes sense for Python).

* [Helm Improvement Proposal] - Helm uses HIPs to track not just code
  enhancements, but they also use it to track suggestions to their goveranance
  model and advisory documentation. The "type" field in the preamble comes from
  Helm.


## Specification

Before creating a PEP, discuss the idea first with the maintainers on our
[forum] and validate that the idea is suitable and welcome. This is to make sure
someone doesn't waste time writing a PEP when a solution already exists, it
doesn't duplicate or overlap with an exiting proposal, it hasn't been proposed
and rejected already, and that the general idea is going in the right direction.


### PEP types

* **Feature** proposals are for changes to the Porter codebase. For example,
  changes to the Porter CLI, libraries, configuration, or mixin/plugin
  protocols. This also includes changes that _remove_ functionality, such as
  deprecating an implemented command.

* **Process** proposals are for changes to how the Porter project is run. For
  example, changes to our project governance, build infrastructure and
  processes, etc.


### PEP Roles

* [Author](#author)
* [Owner](#owner)
* [Reviewer](#reviewer)
* [Commenter](#commenter)


#### Author

Contributors writing the PEP and implementation. This can be a group effort. 💪


#### Owner

Single author who is responsible for moving the PEP forward, collecting
feedback, and gaining consensus with the community. They submit pull requests to
keep the PEP up-to-date, for example updating the status or recording answers or
changes to the design. The owner is the main point of contact for the PEP.

It is the owner's responsibility to follow the PEP process and work towards a
resolution, not the reviewer(s).


#### Reviewer

Maintainer(s) who help get the PEP to match our proposal requirements, ask
questions, and ensure that the PEP has been properly considered before final
approval or rejection. The particular maintainers depends on the affected areas,
scope of change, capacity, etc. At least one reviewer is assigned to coordinate
with the Owner as the PEP moves through the process.


#### Commenter

Community members are welcome to join in on the discussion. Commenters may ask
clarifying questions, provide additional use cases, identify edge cases, etc.


### When is a PEP required?

A PEP is helpful when we need to have an agreed upon design, analysis of
risk/scope, or evaluation of user experience up-front. Based on that a PEP is
required when altering:

* CLI syntax

* Configuration syntax (such as the porter.yaml or config.toml)

* Communication between Porter and mixins/plugins

* Essential behavior of Porter (such as defaults or anything that significantly
  impacts what a mixin developer, bundle author or end-user needs to know)

In general it isn't necessary for bug fixes, documentation updates, website
tweaks, routine build upkeep or [meta] project concerns. For example, correcting
a typo in the PEP documentation can be made directly with a pull request, however
adding a new requirement to the process would require a new PEP. A maintainer
may require a PEP when they feel that a change would benefit from the PEP
process.


### PEP Contents

Proposals are stored in this repository in the [pep](/pep/) directory. They
should be named `NNN-title.md` where `NNN` is the PEP number and `title` is the
lower-case title of the PEP with hyphens in place of spaces. For example, the
filename for this PEP is `001-initial-pep-process.md`.

Every PEP should have the following sections:

1. [Preamble](#preamble)
1. [Abstract](#abstract)
1. [Motivation](#motivation)
1. [Rationale](#rationale)
1. [Specification](#specification)
1. [Backwards Compatibility](#backwards-compatibility)
1. [Security Implications](#security-implications)
1. [Rejected Ideas](#rejected-ideas)
1. [Open Questions](#open-questions)


#### Preamble

Metadata about the PEP, including:

* A short descriptive title
* Number
* Type
* Status
* The github usernames of the author(s)
* The github username of the owner
* The date the PEP was created


#### Abstract

A short (~200 word) description of the technical issue being addressed.


#### Motivation

It should clearly explain why Porter's existing functionality is inadequate to
address the problem that the PEP solves and identify the impacted audience(s) (mixin
developers, bundle authors, end-users). PEP submissions without sufficient
motivation may be rejected outright. This is the most important part at the
beginning and is required before moving forward.


#### Rationale

The rationale fleshes out the specification by describing why particular design
decisions were made.  It should describe alternate designs that were considered
and related work.

The rationale should provide evidence of consensus within the community and discuss important objections or concerns raised during discussion.


#### Specification

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


#### Backwards Compatibility

All PEPs that introduce backwards incompatibilities must include a section
describing these incompatibilities and their severity.  The PEP must explain how
to deal with these incompatibilities, possibly with defaulting or migrations.
PEP submissions without a sufficient backwards compatibility treatise may be
rejected outright.


#### Security Implications

If there are security concerns in relation to the PEP, those concerns should be
explicitly written out to make sure reviewers of the PEP are aware of them.


#### Rejected Ideas

Throughout the discussion of a PEP, various ideas will be proposed which are not
accepted. Those rejected ideas should be recorded along with the reasoning as to
why they were rejected. This both helps record the thought process behind the
final version of the PEP as well as preventing people from bringing up the same
rejected idea again in subsequent discussions.


#### Open Questions

While a PEP is in draft, questions can come up which warrant further discussion.
Those questions should be recorded here so people know that they are being
thought about but do not have a concrete resolution. This helps make sure all
for the PEP to be ready for consideration are complete and reduces
and reduces people duplicating prior discussion. When possible, link the
question to where it is being discussed, such as a [forum] post/comment.


### PEP Lifecycle

We are using the process from Kubernetes Enhancement Proposal for our statuses
because they work better for our implementation than Python (Porter isn't a spec
like python, so Kubernetes process fits us better).

This is the happy path, where a PEP is accepted.

1. The idea is posted to the Porter [forum] where the maintainers and community
   can see it and comment. A maintainer will let you know if it is suitable for
   a PEP and assign it a number.

1. A pull request for the PEP is submitted with an initial status of
   `provisional` in the preamble. The status is **PROPOSED** while the pull
   request is unmerged, and it is not provisional until the pull request is
   merged.

1. Reviewer(s) are assigned to the PEP.

1. Reviewer(s) work with the author(s) to get the PEP to address all sections.
   The implementation should be not started at this time but otherwise the
   proposal should address all other sections.

1. The initial PEP pull request is merged. The status is **PROVISIONAL**.

1. The owner notifies the [mailing list] about the PEP. This is the community's
   chance to discuss the PEP, ask questions, provide their own use cases, and
   participate in the design.

1. The owner continues to edit the PEP as new open issues are raised, decisions
   made, and any changes to the specification are recorded.

1. After the design has settled, the owner submits a PR to change the status to
   `implementable`.

1. Reviewer(s) evaluate the PEP and ensure that due diligence has been followed
   and all pertinent concerns addressed. When the reviewer(s) agree that the
   proposal is ready to implement, the pull request is merged. The status is now
   **IMPLEMENTABLE**. 

1. Author(s) submit a pull request to the relevant Porter repositories
   implementing the proposal.

1. Reviewer(s) evaluate the implementation. In addition to determining if the
   implementation is acceptable, they should to identify gaps in the PEP, for
   example the scope of change was much larger than originally thought, the
   proposed implementation wasn't actually possible as specified or has
   problems, etc.
   
   The pull request may be closed so the author(s) can go back and iterate on
   the PEP. We should be careful about follow-up pull requests at this stage so
   that a PEP isn't incompletely implemented. Critical changes should be made
   together to limit follow-ups to unrelated changes that are not relevant to the PEP.

1. After the implementation is merged, the owner submits a pull request to
   change the status to `implemented`.

1. Reviewer(s) confirm that the PEP has been completely implemented and no
   additional work remains, and then merge the pull request. The final state is
   now **IMPLEMENTED**.


#### End States

These are states that terminate the PEP process:

* A PEP that successfully completes the proposal process is **IMPLEMENTED**.

* The PEP may be rejected at certain points by the reviewer. Reviewer should
  explain the rejection in the active pull request and ultimately close the pull
  request unmerged. The reviewer submits a pull request updating the status to
  `rejected` and reiterate the rejection, including enough context that someone
  can understand it without going back to old pull requests. The status is
  **REJECTED**.

* The owner can withdraw the PEP by submitting a pull request changing the
  status to **WITHDRAWN**.

* When progress on the PEP ceases but we aren't ready to reject it, the PEP can
  be deferred. For example when people are busy for months but they would
  like to come back to it later or we want to do it but now isn't the right time
  because other things need to be done first. In this case the owner or reviewer
  can submit a pull request changing the status to **DEFERRED**.

  When people are ready to work on it again, the owner should submit a pull
  request changing the status back to an appropriate state to continue, such as
  `provisional` or `implementable`.

* Once a PEP has been merged, and a new PEP is started to replace it, or
  supersede it, a pull request should be opened changing the status to
  `replaced` and include a link to the new PEP. If the original PEP was
  implemented, wait to update it to `replaced` until the new PEP is implemented.
  For in-progress PEPs, they can be changed to `replaced` immediately. The
  status is now **REPLACED**.

---


## Implementation

When the PEP process is implemented, the contents of the specification section
will be put into the [CONTRIBUTING.md](/CONTRIBUTING.md) of this repository and
Porter documentation updated to link to the new PEP process. A PEP template will
be added to `pep/000-PEP-TEMPLATE.md` as well.


## Backwards Compatibility

We currently do not have a formal process, but most of these due dilligence
activities are based on what maintainers have been doing behind-the-scenes
today. So backwards compatibility is not a concern.


## Security Implications

None.


## Rejected Ideas

None.


## Open Questions

* How can people filter PEPs by their status?


[Python Enhancement Proposal]: https://github.com/python/peps/blob/master/pep-0001.txt
[Helm Improvement Proposal]: https://github.com/helm/community/blob/master/hips/hip-0001.md
[Kubernetes Enhancement Proposal]: https://github.com/kubernetes/enhancements/blob/master/keps/0001-kubernetes-enhancement-proposal-process.md
[meta]: https://github.com/getporter/porter/labels/meta
[forum]: https://porter.sh/forum
[mailing list]: https://porter.sh/mailing-list
