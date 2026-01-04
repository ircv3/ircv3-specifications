---
title: "Network Icon"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Simon Ser"
    period: "2025"
    email: "contact@emersion.fr"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `ICON` ISUPPORT name. Instead, implementations SHOULD use the `draft/ICON` ISUPPORT name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use unprefixed ISUPPORT names.

## Description

The `NETWORK` ISUPPORT token allows servers to advertise their network name. This specification adds a new ISUPPORT token to advertise a network icon.

## The `draft/ICON` ISUPPORT token

If a server publishes a `draft/ICON` ISUPPORT token, the value MUST be a URL to an image. The URL SHOULD use the "https" scheme.

Servers MAY implement [extended-isupport](extended-isupport.html) to allow clients to fetch the network icon before connection registration.

## Example

    S: :irc.example.org 005 * NETWORK=Example draft/ICON=https://example.org/icon.svg :are supported by this server
