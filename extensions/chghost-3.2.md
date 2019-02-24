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

When the capability is not enabled for clients who share the same channel,
servers should fall back to having the client `QUIT` the server, rejoin all
channels, and re-establish the channel and user modes the client had before, as
though the client had reconnected.

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

## Errata

* Previous versions of this specification used confusing descriptions and have
since been rewritten to include a simpler description and example.
* Previous versions of this specification did not specify whether or not the
client whose user or host changed should receive the message.
