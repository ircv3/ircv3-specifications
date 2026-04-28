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
    name: "internet-catte"
    period: "2026"
    email: "catte@libera.chat"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `ICON` ISUPPORT name. Instead, implementations SHOULD use the `draft/ICON` ISUPPORT name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use unprefixed ISUPPORT names.

## Description

The `NETWORK` ISUPPORT token allows servers to advertise their network name. This specification adds a new ISUPPORT token to advertise a network icon.

## The `draft/ICON` ISUPPORT token

If a server publishes a `draft/ICON` ISUPPORT token, the value MUST be a URL to an image. The URL SHOULD use the "https" scheme.

Servers MAY implement [extended-isupport](extended-isupport.html) to allow clients to fetch the network icon before connection registration.

Servers MAY support requesting different icons using [HTTP client hints], including [`Sec-CH-Prefers-Color-Scheme`] for colour scheme-optimised variants and [`Sec-CH-Prefers-Reduced-Motion`] for non-animated variants.

## Examples

    S: :irc.example.org 005 * NETWORK=Example draft/ICON=https://example.org/icon.svg :are supported by this server

Server supporting light and dark colour scheme icons:

    S: :irc.example.org 005 * NETWORK=Example draft/ICON=https://example.org/icon.png :are supported by this server

    C: GET /icon.png HTTP/1.1
    C: Host: example.org

    S: HTTP/1.1 200 OK
    S: Content-Type: image/png
    S: Accept-CH: Sec-CH-Prefers-Color-Scheme
    S: Vary: Sec-CH-Prefers-Color-Scheme
    S: (default image)

    C: GET /icon.png HTTP/1.1
    C: Host: example.org
    C: Sec-CH-Prefers-Color-Scheme: "dark"

    S: HTTP/1.1 200 OK
    S: Content-Type: image/png
    S: (image optimised for dark backgrounds)

[HTTP client hints]: https://datatracker.ietf.org/doc/html/rfc8942
[`Sec-CH-Prefers-Color-Scheme`]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Sec-CH-Prefers-Color-Scheme
[`Sec-CH-Prefers-Reduced-Motion`]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Sec-CH-Prefers-Reduced-Motion
