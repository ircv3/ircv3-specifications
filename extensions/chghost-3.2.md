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

## Introduction

The `chghost` client capability allows servers to directly inform clients about
clients changing their host and/or user without having to simulate the client
disconnecting. This is useful for servers implementing virtual hosts or masks
and helps with reducing the amount of information spent on a client's UI.

## The `chghost` capability

When a client changes their user and/or host, servers MUST send the `CHGHOST`
message to clients who have enabled the `chghost` capability and either share
the same channel as the target client or have the client on a `MONITOR` list.
Servers SHOULD additionally send the `CHGHOST` message to the client whose
user and/or host has changed if the client supports the `chghost` capability.
The server MUST defer sending CHGHOST messages to the client about successfully
changing it's own user or host (for example, if using SASL and a virtual host
is set) until both a successful `NICK` command and a successful `USER` command
have been received.

When the capability is not enabled for other clients who share channels with or
monitor the changed client, servers SHOULD send messages to simulate the client
reconnecting. This allows clients to keep their user state up to date. For
shared channels, the simulated events SHOULD include appropriate QUIT, JOIN and
MODE commands, to restore membership and user channel modes. For monitored
clients, the events SHOULD include appropriate RPL_MONOFFLINE and RPL_MONONLINE
numerics.

The server MUST send a `CHGHOST` message to a client, but MUST defer doing so
until both a successful `NICK` command a successful `USER` command are received
by the server. The server CAN choose to defer it until after registration is
completed.

## The `CHGHOST` message

The `CHGHOST` message is as follows:

    :nick!user@old_host.local CHGHOST new-user new_host.local

The `new-user` parameter represents the user's "username" or "ident" which may
or may not have changed in the CHGHOST process.

The `new_host.local` parameter represents the new hostname for the user which
may or may not have changed in the CHGHOST process.

## Fallback for not supporting the `chghost` capability

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
* Previous versions of this specification did not specify that the `CHGHOST`
command should be sent after both a valid NICK and a valid USER command have
been received.
* Previous versions of this specification used confusing descriptions and have
since been rewritten to include a simpler description and example.
* Previous versions of this specification did not specify whether or not the
client whose user or host changed should receive the message.
