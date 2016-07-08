# How to Contribute

There are a few different ways that you can contribute to the IRCv3 Working Group:

* **Bug Fixes and Changes**: These fix bugs and issues with existing specifications, with errata updates where necessary.
* **Ideas**: These outline new features that have not yet been worked out in full and do not have a reference implementation.
* **Feature Proposals**: These outline new features and capabilities for the IRC protocol.

**REMINDER: Those with access to the IRCv3 repository should submit PRs from their *personal* forks rather than this repository.** This is to avoid prematurely implying consensus.

## Bug Fixes and Changes

A bug fix may be relatively minor, such as [#186](https://github.com/ircv3/ircv3-specifications/pull/186), or may fix issues found with the specification after it has been merged or ratified, such as [#204](https://github.com/ircv3/ircv3-specifications/pull/204).

Minor bug fixes, such as fixing spelling mistakes or making formatting more consistent, may be submitted without much process. However, try to minimise the number of fixes you submit, as having 13 separate PRs to fix a few spelling and formatting issues can become grating to other participants and observers of the IRCv3 WG.

Submitting bug fixes and changes of this type does not have any stakeholder requirements.

Fixes and changes which address the actual behaviour or meaning of a specification *after they have been released*, such as [#204](https://github.com/ircv3/ircv3-specifications/pull/204) generally require 'errata updates'. Examples of errata updates can be found in various specifications, but they basically leave a log as to how a specification has been changed after it was released. Before submitting a fix of this type, you should talk with other members on the [#ircv3](http://ircv3.net/contact.html) channel and ensure there is support for your change.

## Ideas

Feature proposals contributed to the IRCv3-Specifications repository require a concrete specification and at least one reference or existing implementation.

If you have an idea or a protocol proposal that doesn't have these things or is not yet at this stage, you should make an issue on the [IRCv3/ideas](https://github.com/ircv3/ideas) repository instead. This repository is intended for working group members, as well as developers and users to be able to suggest ideas that are not yet full realised or without a reference implementation.

Contributing to the [IRCv3/ideas](https://github.com/ircv3/ideas) repository does not have any stakeholder requirements. It was created so that contributors may decide to only review ideas that have already been specified and have support from other stakeholders of the IRCv3 WG, or review those that aren't yet at that stage.

## Feature Proposals

A new feature proposal includes things like [`account-tag`](http://ircv3.net/specs/extensions/account-tag-3.2.html), [`cap-notify`](http://ircv3.net/specs/extensions/cap-notify-3.2.html) and [`echo-message`](http://ircv3.net/specs/extensions/echo-message-3.2.html).

Contributing a proposal to the IRCv3-Specifications repo requires a few things. These include:

### Stakeholder Support

Specifications submitted to this repository should be proposed or sponsored by a stakeholder of the IRCv3 Working Group.

### Concrete Specification

Your specification should describe all behaviour that is intended to be implemented by it before being submitted to this repo. If it is not, it may be sent it to the [IRCv3/ideas](https://github.com/ircv3/ideas) repository to be workshopped until it is able to be fully specified.

The protocol syntax specified in your proposal may, of course, be changed after it is submitted. However, it should *not* be submitted with the attitude of *"Well, I didn't think about this too hard and I don't think it's great, what do other people think?"*. Proposals of that type should continue being workshopped with the [IRCv3/ideas](https://github.com/ircv3/ideas) process before being submitted to this repository.

### Reference Implementation

Before being merged into the repository, your proposal must be implemented by at least one piece of IRC software. This is an IRC client, server, bouncer, or preferrably one of each. This implementation *must* be either downloadable or accessible online (so it can be tested by other interested members).

If your proposal is accepted, in the course of drafting an IRCv3 release it will require wider implementation.

### Existing Implementations

If a feature specifies capabilities that already exist in the IRC ecosystem, such as [`monitor`](http://ircv3.net/specs/core/monitor-3.2.html), it should be ensured that any proposal of this type is backwards-compatible with existing implementations.

## Ratifying Feature Proposals

Before being ratified as a published IRCv3 standard, feature proposals must satisfy further requirements. These requirements are intended to ensure the specifications we publish have been tested to a sufficient degree and that they are implementable by real networks and clients without any major bugs or oversights.

These requirements include:

* At least two server (or one bouncer, in the case of specifications specifically intended for IRC bouncers) implementations.
* At least one client implementation.
* Stakeholders of the Working Group to have looked over and approved the proposal.
