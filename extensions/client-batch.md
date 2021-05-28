---
title: "client-initiated batch Extension"
layout: spec
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

## Capabilities

This specification does not introduce any capability.

Clients SHOULD NOT send `BATCH` commands to servers, unless they negotiated
a capability that allows them to do so.

## The client-to-server `BATCH` verb

This specification introduces client initiated batches.
These use the same syntax as server initiated batches.

Once a client has opened a batch, it MUST NOT send any messages
that are not part of the batch.


[`batch`]: ../extensions/batch.html
[`draft/multiline`]: ../extensions/multiline.html
