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

Clients can use the [SASL extension](sasl-3.1.html) to authenticate against IRC servers. Some SASL mechanisms are client-first, ie. the client sends an initial response to start the authentication exchange.

This extension updates the first `AUTHENTICATE` command to accept an initial response, to avoid a roundtrip in the SASL exchange.

## Implementation

Servers supporting this extension MUST advertise the `sasl-ir` capability. Servers MUST accept the new form of the `AUTHENTICATE` command even if the capability hasn't been explciitly enabled by the client.

The initial `AUTHENTICATE` command is extended to accept an extra argument for the initial response:

    AUTHENTICATE <mechanism> [initial-response]

The initial response is optional. When provided, the initial response MUST be encoded as defined by SASL 3.1 (base64 or `+` with a maximum size of 400 bytes). If the initial response is larger than 400 bytes, the client MUST send separate `AUTHENTICATE` commands as usual with the rest of the chunks.

An `AUTHENTICATE` command with an initial response is equivalent to the following exchange:

    C: AUTHENTICATE <mechanism>
    S: AUTHENTICATE +
    C: AUTHENTICATE <initial-response>

## Example protocol exchange

    C: AUTHENTICATE PLAIN amlsbGVzAGppbGxlcwBzZXNhbWU=
    S: :irc.example.org 900 emersion emersion!emersion@emersion.fr emersion :You are now logged in as emersion
    S: :irc.example.org 903 emersion :SASL authentication successful
