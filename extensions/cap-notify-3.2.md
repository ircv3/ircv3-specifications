---
title: IRCv3.2 `cap-notify` Extension
layout: spec
extends:
  - cap-3.1
  - cap-3.2
copyrights:
  -
    name: "Attila Molnar"
    period: "2014"
    email: "attilamolnar@hush.com"
---
## Description

This client capability MUST be named `cap-notify`.

The `cap-notify` client capability allows a client to be notified when
the client capability offering (i.e. capabilities listed in `CAP LS`)
on a server changes.

When enabled, the server MUST notify clients about all new capabilities
and about existing capabilities that are no longer available via the messages
specified in this document.

## Implicit vs explicit support

The `cap-notify` capability MUST be implicitly enabled if the client requests
`CAP LS` with a version of 302 or newer (`CAP LS 302`), as described in the
[capability negotiation 3.2 specification](../core/capability-negotiation-3.2.html).
Further, the `cap-notify` capability MAY NOT be disabled if the client requests
`CAP LS` with a version of 302 or newer.

When implicitly enabled via this mechanism, servers MAY list the `cap-notify` capability
in `CAP LS` and `CAP LIST` responses. Additionally client MAY request the capability with
`CAP REQ`, and capable servers MUST accept and `CAP ACK` the request without side effects.

Clients that do not support `CAP LS` version 302 MAY request the `cap-notify` capability
explicitly. Such clients MAY disable the capability at any time.  This to ease the
adaptation of new features.

## Subcommands

This specification introduces two new CAP subcommands: `NEW` and `DEL`.

Clients implementing this specification SHOULD react to the `CAP NEW` message
with a `CAP REQ` message if they would like to use one of the newly offered
capabilities.

Upon receiving a `CAP DEL` message, the client MUST treat the listed
capabilities cancelled and no longer available.
Clients SHOULD NOT send `CAP REQ` messages to cancel the capabilities in
`CAP DEL`, as they have been canceled already by the server.

Servers MUST cancel any capability-specific behavior for a client after
sending the `CAP DEL` message to the client.

Servers SHOULD NOT remove support for in-use sticky capabilities.

Clients MUST gracefully handle situations when the server removes support
for any capability, including sticky capabilities. If the client cannot
continue to operate without a capability, disconnecting with an appropriate
QUIT message is acceptable.

The format of the new messages is as follows:

    CAP <nick> NEW :<extension 1> [<extension 2> ... [<extension n>]]
    CAP <nick> DEL :<extension 1> [<extension 2> ... [<extension n>]]

The last parameter in both new messages contain a list of new
extensions that became available on the server (in the case of `CAP NEW`)
or a list of extensions that are no longer available (for `CAP DEL`).
The capability list is space separated.
If capability negotiation 3.2 was used, extensions listed MAY contain values.

## Examples

1. Message indicating that a new extension named `batch` is now available:

        :irc.example.com CAP modernclient NEW :batch

2. Message indicating that multiple extensions are no longer available:

        :irc.example.com CAP modernclient DEL :userhost-in-names multi-prefix away-notify

3. Example showing a client requesting capabilities when they become available:

        Server: :irc.example.com CAP tester NEW :away-notify extended-join
        Client: CAP REQ :extended-join away-notify
        Server: :irc.example.com CAP tester ACK :extended-join away-notify

4. Server changes list of supported SASL mechanisms. This example requires
   client to support both `CAP LS 302` and `batch`:

        Client: CAP LS 302
        Server: :irc.example.com CAP * LS :sasl=PLAIN batch
        Client: CAP REQ batch
        ...
        Server: :irc.example.com BATCH +cap example.com/cap-value
        Server: @batch=cap :irc.example.com CAP client DEL :sasl
        Server: @batch=cap :irc.example.com CAP client NEW :sasl=PLAIN,EXTERNAL
        Server: :irc.example.com BATCH -cap

## Errata

* Earlier version of this spec mistakenly missed `<nick>` between `CAP` and
  `NEW`/`DEL` subcommands, but had it in the examples anyway.

* Earlier version of this spec did not mention whether `NEW` and `DEL` can have values or not.

* Earlier versions of this spec were missing clarification of client and server behaviour
  when the capability is implicitly enabled with `CAP LS` version 302 or newer.
