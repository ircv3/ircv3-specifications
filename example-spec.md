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

New capability and tag names should be listed in the _"Notes for implementing work-in-progress version"_ section, with the appropriate disclaimer and `draft/` prefix, until it's ratified.

These keys can be used in the header to specify this spec's relationship to other IRCv3 specifications:

- `updates`: To follow this specification, read the linked spec(s) and then this one.
- `updated-by`: After reading this specification, read the linked spec(s) to get the latest version.
- `extends`: This specification is an optional extension of the linked spec(s).
- `extended-by`: The linked spec(s) are optional extensions to this specification.
- `obsoletes`: Ignore the linked specification(s), as this one replaces them entirely.
- `obsoleted-by`: Ignore this specification, as the linked one(s) replace it entirely.

Each of these links to other specifications, written as such:

    updates:
        - message-tags-3.2
        - message-tags-3.3
    obsoletes:
        - metadata-3.2

The spec names to use here are those specified in the site [specification registry file](https://github.com/ircv3/ircv3.github.io/blob/master/_data/specs.yml).

---


## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `example-spec` capability name. Instead, implementations SHOULD
use the `draft/example-spec` capability name to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.


## Introduction

This feature is intended to solve problem A, or provide a new feature B.

We've written this specification because of C.

In other words, this section contains a rough overview of why the spec exists, the new features you're trying to
provide and/or the issues you're trying to solve. In other words, why other people should be
motivated to implement it, and why you have been motivated to write it.

If this spec is intended for a certain audience (say, mostly bouncers or private servers,
rather than normal public networks), feel free to mention it here. This gives spec readers
an opportunity to think about whether this feature is right for their software and networks.

If there is only a single section in your spec, this should be named "Description" instead of "Introduction".
You may also choose to separate out your 'motivation' and 'introduction/description' into two separate sections.


## Architecture

This specification introduces various capabilities, messages, numerics and tokens as
necessary. This section (and subsections) describe exactly how those work, and define those
new or extended caps/messages/numerics/tokens.

The following sections may be useful to present, as necessary:

### Capabilities

New capabilities that this spec defines are listed here. If there are none, this section can be omitted.

### Tags

New tags that this spec defines are listed here. If there are none, this section can be omitted.

### Messages

New IRC messages (i.e. `PRIVMSG`/`PING`/`CHGHOST`) that this spec defines are listed here. If there are none, this section can be omitted.

### Numerics

New numerics that this spec defines are listed here. For example:

| No. | Label          | Format                                                                  |
| --- | ---------------| ----------------------------------------------------------------------- |
| 001 | `RPL_WELCOME`  | `<client> :Welcome to the Internet Relay Network <nick>!<user>@<host> ` |
| 002 | `RPL_YOURHOST` | `<client> :Your host is <servername>, running version <version> `       |

If there are no new numerics, this section can be omitted.

### RPL_ISUPPORT Tokens

`RPL_ISUPPORT` tokens that this spec defines are listed here. If there are none, this section can be omitted.

### Examples

Examples of this spec in action, for software authors to compare their implementation to.

For example:

    C: CREATE BOX asdf
    S: 300 dan asdf :Box created

If there are multiple clients, you may find it useful to give a short description of what the
example represents and use the following format instead:

    C1 - C: @draft/label=abc PRIVMSG nick :Hello
    C1 - S: @draft/label=abc;msgid=9tohry :dan!d@127.0.0.1 PRIVMSG nick :Hello

    C2 - S: @msgid=9tohry :dan!d@127.0.0.1 PRIVMSG nick :Hello

Where `C1 -` refers to the client/server-sent messages on C1's connection, and `C2 -` refers
to the same thing on C2's connection.


## Implementation Considerations

This section notes specific things software authors will need to look out for and/or check
while implementing your specification. The following introduction may be useful here:

This section notes considerations software authors will need to take into account while
implementing this specification. This section is non-normative.

**Note:** Non-normative means that implementations aren't bound to follow the words in this
section. As well, where the words in this section and _normative_ sections conflict,
implementations should follow the normative section. Unless you say otherwise, all text in
your specification is considered normative.

Where there aren't any implementation considerations, this section can be omitted.


## Security Considerations

New features may have security-specific considerations that authors should take note of
while implementing your specification. This section lists those considerations in one place,
so implementors can look over them.

You must think about whether there are any security considerations for your spec. If no
security considerations exist, this section can be omitted.


## Alternatives

Sometimes, specifications have alternate solutions already out there and being used, or
there are alternate proposals which cover roughly the same features and fixes. This section
can be used to list those alternate proposals and features, note why they have not been used
instead, and why this spec is better than them.

In a large number of cases, this section can be omitted.
