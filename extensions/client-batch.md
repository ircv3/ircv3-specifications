---
title: "client-initiated batch Extension"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Val Lorentz"
    period: "2021"
    email: "progval+ircv3@progval.net"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

All specifications depending on this specification MUST be draft specification
and use only capability names prefixed by `draft/`

## Introduction

This specification extends the [`batch`][] specification to allow
client-initiated BATCH commands.

This specification itself does not introduce any client-to-server
batch type, but is designed as a framework for other specifications.

## Motivation

Certain specifications call for clients to be able to send batched messages to
servers, for instance [`draft/multiline`][]

## Architecture

### Dependencies

This specification does not depend on the [`batch`][] capability,
which is used for server-initiated batches only.

This specification uses the [standard replies][] framework.

### Capabilities

This specification does not introduce any capability.

Clients SHOULD NOT send `BATCH` commands to servers, unless they negotiated
a capability that allows them to do so.

### The client-to-server `BATCH` verb

This specification introduces client initiated batches.
These use the same syntax as server initiated batches.

Once a client has opened a batch, it MUST NOT send any messages
that are not part of the batch, until it is closed
(with `BATCH -reference-tag`).

Clients MUST NOT send nested batches, i.e. clients may not send `BATCH`
commands with a `batch` tag.

### Errors

Servers MUST use `FAIL` messages from the [standard replies][] framework
to notify clients of errors with client-initiated batches.
The command is `BATCH` and the following codes are defined:

* `TIMEOUT <reference-tag>`: the batch was left open for too long,
  all past and future messages in this batch will be ignored.

## Implementation considerations

This section is non-normative.

While this version of the client-initated batch framework does not allow
nested batches, future extensions may allow them.
Client and server implementations may want to take this into account
while designing their internal APIs.


[`batch`]: ../extensions/batch.html
[`draft/multiline`]: ../extensions/multiline.html
