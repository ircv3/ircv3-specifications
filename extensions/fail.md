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

The `<command>` parameter is the primary way that clients discover which of their commands has resulted in an `FAIL` response. In addition, this message may be given a [label](./labeled-response.html) corresponding with the related command, if one is provided by the client. This parameter is case-insensitive. If the `FAIL` message was not triggered by a particular command sent by the client, `*` indicates that this is the case.

The `<argument>` parameters are fairly freeform and simply give more context to how and where the error occurred. These SHOULD stay the same across server and services vendors, but are primary aimed at assisting human debugging of a particular issue. These parameters are case-insensitive.

The `<error-code>` parameter is a case-insensitive machine-readable error code, consistent across vendors and different software, that is the primary way (along with the `<command>`) for clients to display nicer messages for particular errors or to enable handling additional responses from the client side. When possible, implementers should converge on error codes or use ones already in use by other software. The typical format for these codes is `WORDS_SEPARATED_BY_UNDERSCORES`.

The `<description>` is a plain-text description of the error for plain users. Implementors should avoid using overly technical language where possible, and instead focus on how users may resolve the problem. For example, _"Passwords must contain lowercase and uppercase letters, and be 8 or more characters"_ is a better description than something like _"Password consistency and strength enumerability (bcrypt internals) check failed"_.


## Examples

The below are non-normative examples showing how a server may wish to respond to various failing situations. For actual `FAIL` usage, check the relevant specification(s) for the below commands.

### Account registration attempt

    Client: ACC REGISTER * mailto:dan@examplecom passphrase :this is my passphrase
    Server: FAIL ACC REGISTER REG_INVALID_CALLBACK :Email address is not valid

In this example, the command is `ACC`, with the argument `REGISTER`. The code is `REG_INVALID_CALLBACK`, and the description tells the user that the email address they've given is invalid.

### Box Stacking

    Client: BOX STACK CLOCKWISE UPRIGHT FROM_START b1,x2,z3
    Server: FAIL BOX STACK CLOCKWISE BOXES_INVALID :Given boxes are not supported

In this example, the command is `BOX`, with the arguments `STACK` and `CLOCKWISE`. The code is `BOXES_INVALID`, and the description tells the user that the boxes they've given are not supported.

### Rehashing

    Client: REHASH
    Server: FAIL REHASH CONFIG_BAD :Could not reload config from disk

In this example, the command is `REHASH`, with no arguments. The code is `CONFIG_BAD`, and the description tells the user that the config could not be reloaded from disk.
