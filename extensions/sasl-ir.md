---
title: SASL initial response
layout: spec
copyrights:
  -
    name: "Simon Ser"
    period: "2023"
    email: "contact@emersion.fr"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `sasl-ir` CAP name. Instead, implementations SHOULD use the `draft/sasl-ir` CAP name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use unprefixed CAP names.

## Introduction

Clients can use the [SASL extension](sasl-3.1.html) to authenticate against IRC servers. Some SASL mechanisms are client-first, i.e. the client sends an initial response to start the authentication exchange.

This extension updates the first `AUTHENTICATE` command to accept an initial response, to avoid a roundtrip in the SASL exchange.

## Implementation

Servers supporting this extension MUST advertise the `draft/sasl-ir` capability. Servers MUST accept the new form of the `AUTHENTICATE` command even if the capability hasn't been explciitly enabled by the client.

The initial `AUTHENTICATE` command is extended to accept an extra argument for the initial response:

    AUTHENTICATE <mechanism> [initial-response]

The initial response is optional. When provided, the initial response MUST be encoded as defined by SASL 3.1 (base64 or `+` with a maximum size of 400 bytes). If the initial response is larger than 400 bytes, the client MUST send separate `AUTHENTICATE` commands as usual with the rest of the chunks (without repeating the mechanism name).

An `AUTHENTICATE` command with an initial response is equivalent to the following exchange:

    C: AUTHENTICATE <mechanism>
    S: AUTHENTICATE +
    C: AUTHENTICATE <initial-response>

## Example protocol exchange

When the initial response fits into the 400 byte limit:

    C: AUTHENTICATE PLAIN amlsbGVzAGppbGxlcwBzZXNhbWU=
    S: :irc.example.org 900 emersion emersion!emersion@emersion.fr emersion :You are now logged in as emersion
    S: :irc.example.org 903 emersion :SASL authentication successful

When the initial response needs to be split into multiple chunks:

    C: AUTHENTICATE PLAIN AGVtZXJzaW9uAEFyY2hpdGVjdG8gc2VkIHNpdCBwcmFlc2VudGl1bSBhbWV0LiBBZGlwaXNjaSBzdW50IGRpc3RpbmN0aW8gZmFjZXJlIHF1aSBjb3JydXB0aS4gU2l0IGF1dCBzaXQgZXNzZSBhc3Blcm5hdHVyLiBBbGlhcyB2b2x1cHRhdGVzIGV0IHZvbHVwdGF0ZSB0ZW5ldHVyIHF1by4gQXQgaWxsdW0gcmVtIHF1aWEgcXVpYSBxdWlzcXVhbSBjdW1xdWUuIElzdGUgdm9sdXB0YXMgbmloaWwgYXV0LiBWb2x1cHRhdGVtIHBhcmlhdHVyIGFjY3VzYW11cyBhdXQuIFZvbHVwdGF0ZSBldCB2ZWwgYXV0ZW0gc2l0IHF1aWEgZXhwbGljYWJvIGVuaW0gZW5pbS4gTGFib3Jpb3NhbSB0b3RhbSBuaXNpIHF1aSBhZCBub2JpcyB0ZW1wb3JhIGRvbG9ydW0uIFBhcmlhdHVyIGV0IGNvbnNlY3RldHVyIHNpbnQuIA==
    C: AUTHENTICATE SXRhcXVlIHJlaWNpZW5kaXMgZXQgZnVnaWF0IGZhY2VyZSB2ZW5pYW0gcmVjdXNhbmRhZSBkb2xvcmVtIGRlc2VydW50LiBUZW5ldHVyIGN1bSBjdWxwYSBhdHF1ZSBhcmNoaXRlY3RvIGFiIG1heGltZS4=
    S: :irc.example.org 900 emersion emersion!emersion@emersion.fr emersion :You are now logged in as emersion
    S: :irc.example.org 903 emersion :SASL authentication successful
