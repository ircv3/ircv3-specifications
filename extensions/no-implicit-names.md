---
title: "`no-implicit-names` Extension"
layout: spec
meta-description: A capability to disable implicit NAME responses on JOIN
work-in-progress: true
copyrights:
  -
    name: "Simon Ser"
    period: "2023"
    email: "contact@emersion.fr"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `no-implicit-names` CAP name. Instead, implementations SHOULD use the `draft/no-implicit-names` CAP name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use an unprefixed CAP name.

This is a work-in-progress specification.

## Description

This document describes the `no-implicit-names` extension. This allows clients to opt-out from the implicit `NAMES` reply servers send after `JOIN` messages.

Some clients don't need to query the list of channel members for all joined channels. Omitting this information can reduce the time taken to connect to the server, especially on mobile devices and when a large number of channels are joined.

## Implementation

The `no-implicit-names` extension introduces the `draft/no-implicit-names` capability. When negotiated, servers MUST NOT send an implicit `NAMES` reply after sending a `JOIN` message. Servers MUST reply to explicit `NAMES` commands sent by the client as usual.
