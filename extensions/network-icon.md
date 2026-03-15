---
title: "Network Icon"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Simon Ser"
    period: "2025"
    email: "contact@emersion.fr"
  -
    name: "Sadie Powell"
    period: "2026"
    email: "sadie@witchery.services"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `ICON` ISUPPORT name. Instead, implementations SHOULD use the `draft/ICON` ISUPPORT name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use unprefixed ISUPPORT names.

## Description

The `NETWORK` ISUPPORT token allows servers to advertise their network name. This specification adds a new ISUPPORT token to advertise a network icon.

## The `draft/ICON` ISUPPORT token

If a server publishes a `draft/ICON` ISUPPORT token, the value MUST be a URL to an image. This image SHOULD be square. The URL SHOULD use the "https" scheme.

The URL MAY contain the `{size}` template variable that clients MUST replace with an integer to request a specific image size in pixels.  Clients MUST treat this template variable as a hint and MUST NOT assume that they will get the exact same size image back as requested.

Servers MAY implement [extended-isupport](extended-isupport.html) to allow clients to fetch the network icon before connection registration.

## Examples

This section is non-normative.

Server using an SVG icon:

    S: :irc.example.org 005 * NETWORK=Example draft/ICON=https://example.org/icon.svg :are supported by this server

Server using a PNG icon that can be scaled using the `{size}` template variable:

    S: :irc.example.org 005 * NETWORK=Example draft/ICON=https://example.net/icon.png?size={size} :are supported by this server
