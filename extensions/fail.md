---
title: "`FAIL` Message"
layout: spec
copyrights:
  -
    name: "Daniel Oaks"
    period: "2019"
    email: "daniel@danieloaks.net"
---

This document specifies the `FAIL` message, intended to provide a clean, consistent interface for sending general errors to clients. We want implementors to be able to provide appropriate error information for newly-developed commands and functions without having to reserve new numerics.

The `FAIL` message may indicate a complete failure to process a given command or function, or simply something about how the processing goes that clients should be aware of.


## Message Format

This message has the following format:

    FAIL <command> [<argument>...] <error-code> <description>

Here's what each parameter means, along with a longer breakdown:

- `<command>`: The particular command that spawned the `FAIL` message, or `*` for messages initiated outside client commands.
- `<argument>`: One or more parameters that give extra information as to where/why the error occurred.
- `<error-code>`: A machine-readable code for the error encountered. Should stay the same across server and services vendors.
- `<description>`: A plain-text description of the error encountered, so that users know what occurred. May differ between server and services vendors

The `<command>` parameter is the primary way that clients discover which of their commands has resulted in an `FAIL` response. In addition, this message may be given a label corresponding with the related command, if one is provided by the client. This parameter is case-insensitive. If the `FAIL` message was not triggered by a particular command sent by the client, `*` indicates that this is the case.

The `<argument>` parameters are fairly freeform and simply give more context to how and where the error occurred. These SHOULD stay the same across server and services vendors, but are primary aimed at assisting human debugging of a particular issue. These parameters are case-insensitive.

The `<error-code>` parameter is a case-insensitive machine-readable error code, consistent across vendors and different software, that is the primary way (along with the `<command>`) for clients to display nicer messages for particular errors or to enable handling additional responses from the client side. The typical format for these codes is `WORDS_SEPARATED_BY_UNDERSTORES`.

The `<description>` is a plain-text description of the error for plain users. Implementors should avoid using overly technical language where possible, and instead focus on how users may resolve the problem. For example, _"Passwords must contain lowercase and uppercase letters, and be 8 or more characters"_ is a better description than something like _"Password consistency and strength enumerability (bcrypt internals) check failed"_.


## Examples

The below are non-normative examples showing how a server may wish to respond to various failing situations. For actual `FAIL` usage, check the relevant specification(s) for the below commands.

### Account registration attempt

    Client: ACC REGISTER * mailto:dan@examplecom passphrase :this is my passphrase
    Server: FAIL ACC REGISTER REG_INVALID_CALLBACK :Email address is not valid

### Rehashing

    Client: REHASH
    Server: FAIL REHASH DATABASE DB_BAD :Could not reload database from disk
