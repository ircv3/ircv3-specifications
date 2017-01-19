---
title: IRCv3.3 Client Capability Negotiation
layout: spec
copyrights:
  -
    name: "J-P Nurmi"
    period: "2015"
    email: "jpnurmi@gmail.com"
---
## New in version 3.3

IRCv3.3 compliant clients MUST use the capability negotiation protocol
version `303`. This document describes the requirements for version 3.3
compliance.

### Self-messages

Clients that receive self-sent `PRIVMSG` and `NOTICE` messages, MUST
treat them the same way as if the client itself would have sent the
message to the target.

The purpose of this requirement is to allow clients receive full chat
history, that may include their own messages, without having to request
the [`echo-message`](../extensions/echo-message-3.2.html) capability
that requires special handling on the client side.

### Message tags

Clients MUST be able to parse messages that contain arbitrary message
tags. In other words, they MUST NOT make hard-coded assumptions which
tags received messages may contain.

For example, a client that has only requested one capability,
[`server-time`](../extensions/server-time-3.2.html), may still receive
messages that contain other tags in addition to the `time` tag. This
greatly simplifies message tag handling for bouncers, because they do
not need to filter out message tags that have not been explicitly
requested by the client. 

## Examples

    Client: CAP LS 303
    Client: NICK jpnurmi
    Client: USER jpnurmi 0 0 :J-P Nurmi
    Server: CAP * LS :extended-join multi-prefix sasl server-time
    Client: CAP REQ :server-time
    Server: CAP ACK :server-time
    [...]
    Server: @example.com/foo=bar;time=2015-08-22T15:19:02.270Z :jpnurmi!jpnurmi@example.com PRIVMSG #ircv3 :hi

In the above example, the server sends a client their own message that
contains an additional tag that the client has not explicitly requested.
