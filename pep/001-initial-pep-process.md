---
title: Initial Porter Enhancement Proposal Process # 50 characters
number: 001
status: proposed
owner: @carolynvs
authors:
  - @carolynvs
created: 2020-10-29
type: meta
---

## Abstract

Let's define a process to propose and evaluate changes to the Porter project.
The Porter Enhancement Proposal (PEP) process is a lightweight mix of a [Python
Enhancement Proposal] and a [Kubernetes Enhancement Proposal]. A PEP is a
document that explains why a change is desirable and proposes how to implement
it.

[Python Enhancement Proposal]: https://github.com/python/peps/blob/master/pep-0001.txt
[Kubernetes Enhancement Proposal]: https://github.com/kubernetes/enhancements/blob/master/keps/0001-kubernetes-enhancement-proposal-process.md

## Motivation

Adopting a proposal process will improve how proposals are handled and increase
transparency with the community. Right now, people make proposals either in an
issue or on our forum or slack messages, etc, we hash it out there, sometimes
making a new issue to track the final set of work, and then start on it.

The hodge-podge nature of our informal process makes it difficult to engage with
these discussions and give people a reasonable turnabout time on feedback. It
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
  template for us to start with. Some changes are necessary though due to the
  differences in Porter vs Python. Porter is an implementation, whereas Python
  is a specification. That is why the Kuberentes and Helm processes were
  incorporated as well.
* [Kubernetes Enhancement Proposal] - The PEP statuses were based on the KEP
  process since it made more sense to say something was "implementable" and
  "implemented" vs. having people write a reference implemenation of a spec
  (which makes more sense for just Python).
* [Helm Improvement Proposal] - Helm uses HIPs to track not just code
  enhancements, but they also use it to track suggestions to their goveranance
  model and advisory documentation. The "type" field in the preamble comes from
  Helm.

## Specification

### PEP Roles

* Author(s) - Contributors writing the PEP and the implementation. It can be a group effort. 💪 
* Owner - Author who is responsible for moving the PEP forward. They submit pull requests to keep the PEP up-to-date, for example updating the status or recording answers or changes to the design, and is the main point of contact for the PEP.
* Reviewer(s) - Maintainer(s) who help get the PEP to match PEP requirements, ask questions and ensure that the PEP has been fully considered before final approval. The particular maintainers depends on the affected areas, scope of change, the lunar cycle, etc. At least one person is assigned to make sure the process moves forward to a resolution.
* Commenter(s): Community members are welcome to join in on the discussion. Ask clarifying questions, provide your own use cases, identify edge cases, etc.

### When is a PEP required?

A PEP is helpful when we need to have an agreed upon design, analysis of risk/scope, or evaluation of user experience up-front. Based on that a PEP is required when altering:

* CLI syntax
* Configuration syntax (such as the porter.yaml or config.toml)
* Communication between Porter and mixins/plugins
* Essential behavior of Porter (such as defaults or anything that significantly impacts what a mixin developer, bundle author or end-user needs to know)

In general it isn't necessary for bug fixes, documentation updates, website tweaks, build changes or [meta](https://github.com/getporter/porter/labels/meta) project concerns. A maintainer may ask for a PEP when they feel that a change would benefit from
the PEP process.

### PEP Contents

Every PEP should have the following sections:

Each PEP should have the following parts/sections (modified from Python's PEP):

1. Preamble -- Meta-data about the PEP, including the PEP number, a short descriptive title (limited to a maximum of 44 characters), the names of the author(s) and status.

2. Abstract -- a short (~200 word) description of the technical issue being addressed.

3. Motivation -- It should clearly explain why Porter's existing functionality is inadequate to address the problem that the PEP solves and the impacted audience(s) (mixin developers, bundle authors, end-users). PEP submissions without sufficient motivation may be rejected outright. This is the most important part at the beginning and is required before moving forward.

4. Rationale -- The rationale fleshes out the specification by describing why particular design decisions were made.  It should
describe alternate designs that were considered and related work.

    The rationale should provide evidence of consensus within the community and discuss important objections or concerns raised during discussion.

5. Specification -- The technical specification should describe the command and/or configuration syntax and semantics of any new feature. 
  * If this is a command, we are looking for what the `porter COMMAND --help` would look like: description of command, arguments, flags, default behavior and error handling.
* If this is a syntax change to a configuration file, define the allowed syntax, at least one example per use case, covering defaults and error handling.
* All PEPs will be reviewed for user experience. So make sure to think about the common use case, how people can accomplish more advanced scenarios, precedence from existing Porter features or other tools in the ecosystem, and how the change fits into Porter workflows and tasks.

6. Backwards Compatibility -- All PEPs that introduce backwards incompatibilities must include a section describing these
incompatibilities and their severity.  The PEP must explain how to deal with these incompatibilities, possibly with defaulting or migrations.  PEP submissions without a sufficient backwards compatibility treatise may be rejected outright.

7. Security Implications -- If there are security concerns in relation to the PEP, those concerns should be explicitly written out to make sure reviewers of the PEP are aware of them.

10. Rejected Ideas -- Throughout the discussion of a PEP, various ideas will be proposed which are not accepted. Those rejected ideas should be recorded along with the reasoning as to why they were rejected. This both helps record the thought process behind the final version of the PEP as well as preventing people from bringing up the same rejected idea again in subsequent discussions.

11. Open Issues -- While a PEP is in draft, ideas can come up which warrant further discussion. Those ideas should be recorded so people know that they are being thought about but do not have a concrete resolution. This helps make sure all issues required for the PEP to be ready for consideration are complete complete and reduces people duplicating prior discussion.

### PEP Lifecycle

We are using the process from Kubernetes Enhancement Proposal for our status's because they work better for our implementation than Python (Porter isn't a spec like python, so Kubernetes process fits us better).

This is the happy path, where a PEP is accepted.

1. A pull request for the PEP is submitted. The status is **PROPOSED**.
1. Use the pull request number to assign a number to the PEP. This number helps people quickly find and refer to PEPs.
1. Reviewer(s) are assigned to the PEP.
1. Reviewer(s) work with the author(s) to get the PEP to address all sections. Do not do the implementation at this time but otherwise should have everything else.
1. PEP is merged. The status is **PROVISIONAL**.
1. The owner notifies the mailing list about the PEP. This is the community's chance to discuss the PEP, ask questions, provide their own use cases, and participate in the design.
1. The owner continues to edit the PEP as new open issues come up, decisions are made and changes to the design are made.
1. After the design has settled, the owner submits a PR to change the status to `implementable`. 
1. Reviewer(s) evaluate the PEP and ensure that due diligence has been followed and all pertinent concerns addressed. If it is acceptable it is merged. The status is now **IMPLEMENTABLE**. 
1. Once the PEP is implementable, author(s) submit a pull request to Porter's repositories that implements the design.
1. Reviewer(s) evaluate the implementation. Beyond determining if the implementation is acceptable, they should use this to identify gaps in the PEP, for example the scope of change was much larger than originally thought, the proposed implementation wasn't actually possible as specified, etc. The pull request may be closed so the author(s) can go back and iterate on the PEP. We should be careful about follow-up pull requests at this stage so that a PEP isn't incompletely implemented, critical changes should be made together. Limit follow-ups to unrelated changes that not relevant to the PEP.
1. When the implementation is merged, the reviewer or owner submits a pull request to change the status to implemented.
1. Reviewer(s) merge the pull request and the final state is **IMPLEMENTED**.

#### Resting States
  * The PEP may be rejected at certain points by the reviewer. Reviewer should explain the rejection in the active pull request and ultimately close the pull request unmerged. The reviewer submits a pull request updating the status to `rejected` and reiterate the rejection, including enough context that someone can understand it without going back to old pull requests. The status is **REJECTED**.
* The owner can withdraw the PEP by submitting a pull request changing the status to **WITHDRAWN**.
* When progress on the PEP ceases but we aren't ready to reject it, for example people are busy for months and but they  would like to come back to it later or we want to do it but now isn't the right time because other things need to be done first, the owner or reviewer can submit a pull request changing the status to **DEFERRED**.
* Once a PEP has been merged, and a new PEP is started to replace it, or supersede it, a pull request should be opened changing the status to `replaced`. If the original PEP was implemented, wait to update it to replaced until the new PEP is implemented. For in-progress PEPs, it can be merged immediately. The status is now **REPLACED**.

## Implementation

When the PEP process is implemented, the contents of the specification section
will be put into the [CONTRIBUTING.md](/CONTRIBUTING.md) of this repository and
the Porter repository updated to link to this repository.

## Backwards Compatibility

We currently do not have a formal process, but most of these due dilligence
activities are based on what maintainers have been doing behind-the-scenes
today. So backwards compatibility is not a concern.

## Security Implications

None.

## Rejected Ideas

None.

## Open Issues

* How can people filter PEPs by their status?

## Porter Enhancement Proposal (PEP) Process

We follow a process called Porter Enhancement Proposal (PEP) for making impactful changes
to the Porter project. 

* Before submitting a PEP, discuss the idea first with the maintainers and make sure they agree that this is something worth pursuing. This is to make sure someone doesn't waste time starting a PEP when a solution already exists, doesn't have a duplicate and that the general idea is going in the right direction. The best place to submit an idea is our forums (here).
* Have a template for PEPs that outlines what a PEP should look like and the sections it should contain, etc.
* After getting a 👍 from maintainers, the proposer drafts a PEP and submits it to our proposals repository (TBD).
* The PEP is reviewed and iterated upon before being marked "implementable" and then is implemented in Porter by the author(s).
