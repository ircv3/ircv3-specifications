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
clients changing their host or user without having to simulate the client
reconnecting. This is useful for servers implementing virtual hosts or masks
and helps reduce clutter on the client's UI.

## The `chghost` capability

When a client changes their user or host, servers MUST send the `CHGHOST`
message to clients who have enabled the `chghost` capability and either share
the same channel as the target client or have the client on a `MONITOR` list.
Servers SHOULD additionally send the `CHGHOST` message to the client whose
user or host has changed if the client supports the `chghost` capability.

When the capability is not enabled for other clients who share channels with or
monitor the changed client, servers SHOULD send messages to simulate the client
reconnecting. This allows clients to keep their user state up to date. For
shared channels, the simulated events SHOULD include appropriate `QUIT`, `JOIN`
and MODE commands, to restore membership and user channel modes. For monitored
clients, the events SHOULD include appropriate `RPL_MONOFFLINE` and
`RPL_MONONLINE` numerics.

When the server sends a `CHGHOST` message to a client, it MUST defer doing so
until both successful `NICK` and `USER` commands have been received by the
server.  The server MAY choose to defer it until after registration is
completed, for example if a valid SASL authentication during client
registration triggers an assignment of a virtual host. If the server does not
defer it until registration is completed, and either the user or the host of
the client changes, the server MUST send a new `CHGHOST` message.

## The `CHGHOST` message

The `CHGHOST` message is as follows:

    :nick!old_user@old_host.local CHGHOST new_user new_host.local

The `new-user` parameter represents the user's "username" or "ident" which may
or may not have changed in the CHGHOST process.

The `new_host.local` parameter represents the new hostname for the user which
may or may not have changed in the CHGHOST process.

If the server chooses to do so, the server can prepend the user with a tilde
(such as denoting that the user was not sent by an ident daemon). When doing
so, the command might look like:

    :nick!old_user@old_host.local CHGHOST ~new_user new_host.local

## Fallback for not supporting the `chghost` capability

    :nick!old_user@old_host.local QUIT :Changing hostname
    :nick!new_user@new_host.local JOIN #ircv3
    :ircd.local MODE #ircv3 +v :nick

Like above, if the server chooses to do so, the server can prepend the user
with a tilde:

    :nick!old_user@old_host.local QUIT :Changing hostname
    :nick!~new_user@new_host.local JOIN #ircv3
    :ircd.local MODE #ircv3 +v :nick

## Errata

* Previous versions of this specification did not specify that the `CHGHOST`
command should be sent after both a valid NICK and a valid USER command have
been received.
* Previous versions of this specification used confusing descriptions and have
since been rewritten to include a simpler description and example.
* Previous versions of this specification did not specify whether or not the
client whose user or host changed should receive the message.
