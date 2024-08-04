---
title: "`extended-isupport` Extension"
layout: spec
copyrights:
  -
    name: "Simon Ser"
    period: "2024"
    email: "contact@emersion.fr"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `extended-isupport` capability name. Instead, implementations SHOULD
use the `draft/extended-isupport` capability name to be interoperable with
other software implementing a compatible work-in-progress version. The final
version of the specification will use unprefixed capability names.

## Introduction

`RPL_ISUPPORT` is used to advertise metadata about the server as key-value
entries. However, `RPL_ISUPPORT` is only sent by servers after connection
registration. This undermines the usefulness of `RPL_ISUPPORT`: some of the
metadata would be useful to clients prior to connection registration. This
extension adds a mechanism to send `RPL_ISUPPORT` messages before connection
registration, as well as an end of `RPL_ISUPPORT` list indication.

## Implementation

### `draft/extended-isupport` capability

When the `draft/extended-isupport` capability is enabled by the client, the
server MUST send one or more `RPL_ISUPPORT` messages (grouped inside a
`draft/isupport` batch if the `batch` capability is enabled). The capability
MAY be enabled by the client before connection registration completes (ie,
before the client sends `CAP END`, and before the server sends `RPL_WELCOME`).

Before connection registration completes, while `draft/extended-isupport` is
enabled, the server MAY send updates to the key-value entries via subsequent
`RPL_ISUPPORT` messages (the same way it would after connection registration
completes without this extension).

The server MAY skip the `RPL_ISUPPORT` replies usually sent when connection
registration completes, if it already sent all information.

Before connection registration, the server MAY send only a subset of the full
`RPL_ISUPPORT` list. In that case, the server MUST send a `RPL_ISUPPORT` list
when connection registration completes with entries previously omitted.

The server MUST send one or more `RPL_ISUPPORT` messages when the capability is
enabled, including after connection registration or if the disables then
re-enables the capability.

### `draft/isupport` batch

The server MUST group all `RPL_ISUPPORT` messages inside a `draft/isupport`
batch when the [`batch`][] and `draft/extended-isupport` capabilities are
enabled. The server MUST NOT send any unbatched `RPL_ISUPPORT` message while
both of these capabilities are enabled. The order in which the capabilities are
enabled is not significant.

The batch MUST only contain one or more `RPL_ISUPPORT` messages, it MUST NOT
contain any other message.

As usual, servers can update or delete existing values by sending additional
`RPL_ISUPPORT` messages in a `draft/isupport` batch after the initial batch.

## Examples

Enabling the capability:

    C: CAP LS 302
    S: :irc.example.org CAP * LS :multi-prefix sasl batch draft/extended-isupport
    C: CAP REQ batch draft/extended-isupport
    S: :irc.example.org CAP * ACK :batch draft/extended-isupport
    S: :irc.example.org BATCH +asdf draft/isupport
    S: @batch=asdf :irc.example.org 005 * NETWORK=Example NICKLEN=30 FOO=bar
    S: :irc.example.org BATCH -asdf
    C: NICK emersion
    C: USER emersion 0 * :Simon Ser
    C: CAP END
    S: 001 emersion :어서 오세요

Sending a change:

    S: :irc.example.org BATCH +asdf draft/isupport
    S: @batch=asdf :irc.example.org 005 * CHANNELLEN=64 NICKLEN=42 -FOO
    S: :irc.example.org BATCH -asdf
