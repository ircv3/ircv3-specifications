---
title: IRCv3 `brb` Extension
layout: spec
extends: resume
copyrights:
  -
    name: "Jess"
    period: "2019"
    email: "jess@jesopo.uk"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `brb` capability name. Instead, implementations SHOULD
use the `draft/brb` capability name to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

Occasionally, clients will need to restart in order to update servers and, at 
the moment, this will cause a `QUIT` to be sent out for the user on all their 
networks (unless the client supports in-place upgrading e.g. weechat.)

With the introduction of the `resume` spec, we have an opportunity to leverage 
the ability to resume an old session to achieve this but we need a way to 
client-initiate a clean disconnect that holds a session open (`resume` is 
designed solely for ping timeouts.)

## Architecture

### Commands

#### `BRB` Command

This command MAY ONLY be sent by a client that has previously negotiated the 
`resume` capability.

    BRB <reason>

`<reason>` is a trailing argument that will be used as an `AWAY` message for the 
client until either they resume the session or the window of time for resumption 
closes - in which case it will be used for a `QUIT` reason from the client.

### Messages

#### Success

    BRB SUCCESS

Used to tell the client that their `BRB` command was successful.

This is to be immediately followed by the server terminating the TCP connection.

#### Failure

    FAIL BRB CANNOT_BRB :<reason>

Used when, for any reason, the server could not accept the `BRB` request.

## Examples

#### Successful request

    C: BRB :software updates!
    S: BRB SUCCESS 
    ... TCP connection is immediately terminated by the server ...

    ... client reconnects and does draft/resume-0.3 handshake ...

#### Failed request

    C: BRB :software updates!
    S: FAIL BRB CANNOT_BRB :BRB is unavailable at this time
    ... IRC connection continues as it was ...

#### Failed request with `QUIT` fallback

    C: BRB :software updates!
    S: FAIL BRB CANNOT_BRB :BRB is unavailable at this time
    C: QUIT :software updates!

