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
use the `draft/extended-isupportt` capability name to be interoperable with
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
server MUST send one or more `RPL_ISUPPORT` messages followed by a
`RPL_ENDOFISUPPORT` message. While `draft/extended-isupport` is enabled, the
server MAY send updates to the key-value entries via subsequent `RPL_ISUPPORT`
and `RPL_ENDOFISUPPORT` messages.

The server MAY skip the `RPL_ISUPPORT` replies usually sent when connection
registration completes, if it already sent all information.

Before connection registration, the server MAY send only a subset of the full
`RPL_ISUPPORT` list. In that case, the server MUST send a `RPL_ISUPPORT` list
when connection registration completes with entries previously omitted.

### `RPL_ENDOFISUPPORT` numeric (370)

    :<server> 370 <client> :<info>

This numeric is used to indicate the end of a `RPL_ISUPPORT` list. Servers MUST
always end any `RPL_ISUPPORT` list with `RPL_ENDOFISUPPORT` when the
`draft/extended-isupport` capability is enabled.

## Examples

Enabling the capability:

    C: CAP LS 302
    S: :irc.example.org CAP * LS :multi-prefix sasl draft/extended-isupport
    C: CAP REQ draft/extended-isupport
    S: :irc.example.org CAP * ACK draft/extended-isupport
    S: :irc.example.org 005 * NETWORK=Example NICKLEN=30
    S: :irc.example.org 370 * :End of ISUPPORT list

Sending a change:

    S: :irc.example.org 005 * CHANNELLEN=64 -NETWORK
    S: :irc.example.org 370 * :End of ISUPPORT list
