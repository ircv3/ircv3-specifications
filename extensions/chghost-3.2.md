---
title: IRCv3.2 `chghost` Extension
layout: spec
copyrights:
  -
    name: "Christine Dodrill"
    period: "2013"
    email: "me@christine.website"
  -
    name: "Ryan"
    period: "2016"
    email: "ryan@hashbang.sh"
---

The chghost client capability allows servers to directly inform clients about a
host or user change without having to send fake quits or joins. This capability
MUST be referred to as `chghost` at capability negotiation time.

When the capability is enabled, servers MUST send the `CHGHOST` message to
clients sharing the same channel as the target client, informing those clients
that the user or host of the target client has changed.

The `CHGHOST` message is as follows:

    :nick!user@old_host.local CHGHOST new-user new_host.local

The `new-user` parameter represents the user's "username" or "ident" which may
or may not have changed in the CHGHOST process.

The `new_host.local` parameter represents the new hostname for the user which
may or may not have changed in the CHGHOST process.

When the capability is not enabled for clients who share the same channel,
servers should fall back to having the client `QUIT` the server, rejoin all
channels, and re-establish the channel and user modes the client had before, as
though the client had reconnected.

    :nick!user@old_host.local QUIT :Changing hostname
    :nick!new-user@new_host.local JOIN #ircv3
    :ircd.local MODE #ircv3 +v :nick

## Examples

In this example, `tim!~toolshed@backyard` gets their username changed to `b` and
their hostname changed to `ckyard`:

    :tim!~toolshed@backyard CHGHOST b ckyard

In this example, `tim!b@ckyard` gets their username changed to `~toolshed` and
their hostname changed to `backyard`:

    :tim!b@ckyard CHGHOST ~toolshed backyard

## Errata

* Previous versions of this specification did not include any examples, which made
it unclear as to whether the de-facto `~` prefix should be included on CHGHOST
messages. The new examples make clear that it should be included.
* Previous versions of this specification used confusing descriptions and have
since been rewritten to include a simpler description and example.
* Previous versions of this specification did not specify whether or not the
client whose user or host changed should receive the message.
