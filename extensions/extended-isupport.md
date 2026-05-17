---
title: "`extended-isupport` Extension"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Simon Ser"
    period: "2024"
    email: "contact@emersion.fr"
  -
    name: "Valerie Pond"
    period: "2026"
    email: "v.a.pond@outlook.com"
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
registration, as well as a dedicated command to request the `ISUPPORT` list and
a batch for `RPL_ISUPPORT` messages.

Additionally, the standard `RPL_ISUPPORT` semantics treat a repeated token as
a replacement, not an extension, which combined with the 512-byte line limit
bounds the size of any single token's value at roughly 400 bytes. This
extension defines an opt-in append form (`KEY+=value`) for list-valued tokens
that allows a server to deliver a value across multiple lines.

## Implementation

### `ISUPPORT` command

A new `ISUPPORT` command is introduced to request the full `RPL_ISUPPORT` list.
When receiving this command, the server MUST reply with one or more
`RPL_ISUPPORT` messages (grouped inside a `draft/isupport` batch if the `batch`
and `draft/extended-isupport` capabilities are enabled, see below).

### `draft/extended-isupport` capability

When the `draft/extended-isupport` capability is enabled by the client, the
server MUST accept `ISUPPORT` commands before connection registration
completes (ie, before the client sends `CAP END`, and before the server sends
`RPL_WELCOME`).

Before connection registration completes, while `draft/extended-isupport` is
enabled, the server MAY send updates to the key-value entries via subsequent
`RPL_ISUPPORT` messages (the same way it would after connection registration
completes without this extension).

The server MAY skip the `RPL_ISUPPORT` replies usually sent when connection
registration completes, if it already sent all information.

Before connection registration, the server MAY send only a subset of the full
`RPL_ISUPPORT` list. In that case, the server MUST send a `RPL_ISUPPORT` list
when connection registration completes with entries previously omitted.

### `draft/isupport` batch

The server MUST group all `RPL_ISUPPORT` messages inside a `draft/isupport`
batch when the [`batch`][batch] and `draft/extended-isupport` capabilities are
enabled. The server MUST NOT send any unbatched `RPL_ISUPPORT` message while
both of these capabilities are enabled. The order in which the capabilities are
enabled is not significant.

The batch MUST only contain one or more `RPL_ISUPPORT` messages, it MUST NOT
contain any other message.

As usual, servers can update or delete existing values by sending additional
`RPL_ISUPPORT` messages in a `draft/isupport` batch after the initial batch.

### Append syntax for list-valued tokens

When the `draft/extended-isupport` capability is enabled, the syntax of an
`RPL_ISUPPORT` token is extended to:

    token       =  ( key "+=" value )           ; append form
                /  ( key "=" value )            ; replacement form
                /  ( key )                      ; flag form
                /  ( "-" key )                  ; deletion form

The `+=` delimiter is treated atomically and MUST NOT be split across two
tokens.

The append form is defined as byte-wise concatenation. The client
performs no interpretation of either the current value or the appended
chunk; it simply appends the bytes of `value` to whatever bytes are
currently held for `key`. Any structural delimiter required by the
token's grammar (such as a separating comma) MUST be included by the
server inside the appended `value`. To extend a non-empty
comma-separated list, for example, the server sends the appended chunk
with a leading comma:

    KEY=foo,bar,baz
    KEY+=,qux,quux,corge

producing the final value `foo,bar,baz,qux,quux,corge`.

The `+=` form has meaning only for `RPL_ISUPPORT` tokens whose value
grammar is explicitly defined to permit it. A token specification that
wishes to permit `+=` MUST state so explicitly, and MAY constrain
behaviour beyond byte-wise concatenation (for example, the semantics of
appending an element that is already present, or how wildcard or
exclusion markers in the token's grammar combine across appended
chunks).

For tokens whose specification does not permit `+=`, a server MUST NOT
send the `+=` form. A client receiving `+=` for a token whose grammar is
unknown to it, or whose specification does not permit `+=`, SHOULD ignore
the append; it MUST NOT interpret it as a replacement.

Processing rules apply token-by-token to the value the client currently
holds:

- `KEY=value` sets the value to `value`, replacing any prior value.
- `KEY+=value` sets the value to the byte-wise concatenation of the
  current value and `value`. If no value is currently held (the token has
  not been advertised in this session, or was deleted), the result is
  simply `value`. Whether the resulting bytes form a well-formed value
  for the token is the token specification's concern, not this one's.
- `-KEY` removes the value entirely; a subsequent `KEY+=value` then
  behaves as in the unset case above.
- `KEY+=` (append form with an empty value) is a no-op.

Multiple `+=` tokens for the same key MAY appear within a single
`RPL_ISUPPORT` reply, across multiple replies, and across batches. The
order of concatenation is the order in which the tokens are received.

A server that can fit a token's value in a single `RPL_ISUPPORT` line
SHOULD continue to use the plain `=` form. A server MUST NOT use `+=` to
clients that have not negotiated `draft/extended-isupport`.

## Examples

Enabling the capability:

    C: CAP LS 302
    S: :irc.example.org CAP * LS :multi-prefix sasl batch draft/extended-isupport
    C: CAP REQ batch draft/extended-isupport
    S: :irc.example.org CAP * ACK :batch draft/extended-isupport
    C: ISUPPORT
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

Delivering a list-valued token (here a hypothetical `EXAMPLELIST`) across
two lines using the append form. Note the leading comma on the appended
chunk: the server is responsible for emitting any separator required by
the token's grammar.

    S: :irc.example.org BATCH +xyz draft/isupport
    S: @batch=xyz :irc.example.org 005 * EXAMPLELIST=foo,bar,baz
    S: @batch=xyz :irc.example.org 005 * EXAMPLELIST+=,qux,quux,corge
    S: :irc.example.org BATCH -xyz

The resulting value held by the client is `foo,bar,baz,qux,quux,corge`. A
later plain `=` resets the accumulation; a later `-EXAMPLELIST` clears
it.

[batch]: ../extensions/batch.html
