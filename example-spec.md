---
title: Specification Name
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Your Name"
    email: "you@yourdomain.com"
    period: "Year Range (e.g. 2010-2016)"
---

This is an example specification, which is intended to help you while writing a new proposal!

Feel free to use this as a rough guideline while writing up your specification. However,
keep in mind that the sections here are not set in stone.

Also take a look at other specifications and the CONTRIBUTING.md file for further
suggestions and information to keep in mind while writing a proposal.

## Introduction

This feature is intended to solve problem A, or provide a new feature B.

We've written this specification because of C.

In other words, this section contains a rough overview of why the spec exists, the new features you're trying to
provide and/or the issues you're trying to solve. In other words, why other people should be
motivated to implement it, and why you have been motivated to write it.

## Architecture

This specification introduces various capabilities, messages, numerics and tokens as
necessary. This section (and subsections) describe exactly how those work, and define those
new or extended caps/messages/numerics/tokens.

The following sections may be useful to present, as necessary:

### Capabilities

New capabilities that this spec defines are listed here.

### Messages

New messages that this spec defines are listed here.

### Numerics

New numerics that this spec defines are listed here. For example:

| No. | Label          | Format                                                                  |
| --- | ---------------| ----------------------------------------------------------------------- |
| 001 | `RPL_WELCOME`  | `<client> :Welcome to the Internet Relay Network <nick>!<user>@<host> ` |
| 002 | `RPL_YOURHOST` | `<client> :Your host is <servername>, running version <version> `       |

### RPL_ISUPPORT Tokens

RPL_ISUPPORT tokens that this spec defines are listed here.

### Examples

Examples of this spec in action, for software authors to compare their implementation to.

For example:

    C: CREATE BOX asdf
    S: 300 dan asdf :Box created

## Security Considerations

New features may have security-specific considerations that authors should take note of
while implementing your specification. This section lists those considerations in one place,
so they can look over them.

If no security considerations exist, you should include this text instead:

There are no security considerations that implementations should note when implementing this
specification.

## Alternatives

Sometimes, specifications have alternate solutions already out there and being used, or
there are alternate proposals which cover roughly the same features and fixes. This section
can be used to list those alternate proposals and features, note why they have not been used
instead, and why this spec is better than them.

In a large number of cases, this section can be omitted.
