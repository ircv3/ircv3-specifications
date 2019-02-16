---
title: IRCv3.2 `chghost` Extension
layout: spec
copyrights:
  -
    name: "Christine Dodrill"
    period: "2013"
    email: "xena@yolo-swag.com"
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

Servers SHOULD also send the `CHGHOST` command to the client who changed their
host, as well as the previously-unspecified `396` numeric, whose exact value
count changes per implementation (from two to three arguments), but always
places the client's new host in the second-to-last position.

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

# Errata

* Previous versions of this specification used confusing descriptions and have
since been rewritten to include a simpler description and example.
* Previous versions of this specification did not specify whether or not the
client whose user or host changed should receive the message.
