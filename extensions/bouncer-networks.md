---
title: IRCv3 bouncer networks extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Darren Whitlen"
    period: "2020"
    email: "darren@kiwiirc.com"
  -
    name: "Simon Ser"
    period: "2021"
    email: "contact@emersion.fr"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `bouncer-networks` CAP names. Instead, implementations SHOULD use
the `draft/bouncer-networks` CAP names to be interoperable with other software
implementing a compatible work-in-progress version. The final version of the
specification will use unprefixed CAP names.

## Description

This document describes the `bouncer-networks` extension. This enables clients
to discover servers that are bouncers, list and edit upstream networks the
bouncer is connected to.

Each network has a unique per-user ID called "netid". It MUST NOT change during
the lifetime of the network. TODO: character restrictions for network IDs.

Networks also have attributes. Attributes are encoded in the message-tag
format.

## Implementation

The `bouncer-networks` extension defines a new `RPL_ISUPPORT` token and a new
`BOUNCER` command.

### `RPL_ISUPPORT` token

The server can advertise a `BOUNCER_NETID` token in its `RPL_ISUPPORT` message.
Its optional value is the network ID bound for the current connection.

### `BOUNCER` command

A new `BOUNCER` command is introduced. It has a case-insensitive subcommand:

    BOUNCER <subcommand> <params...>

#### `BIND` subcommand

The `BIND` subcommand selects an upstream network to bind to for the lifetime
of the current connection. Clients can only send it after authentication but
before the registration completes.

    BOUNCER BIND <netid>

#### `LISTNETWORKS` subcommand

The `LISTNETWORKS` subcommand queries the list of upstream networks.

    BOUNCER LISTNETWORKS

The server replies with:

    BOUNCER LISTNETWORKS <netid> <attributes>

And then ends the list (even if empty) with:

    BOUNCER LISTNETWORKS *

### Errors

Errors are returned using the standard replies syntax. The general syntax is:

    FAIL BOUNCER <code> <subcommand> [context...] <description>

If a client sends a `BIND` subcommand before authentication:

    FAIL BOUNCER ACCOUNT_REQUIRED BIND Authentication required

If a client sends a `BIND` subcommand after registration:

    FAIL BOUNCER REGISTRATION_IS_COMPLETED BIND Cannot bind to a network after registration

If a client sends a `BIND` subcommand with an invalid network ID:

    FAIL BOUNCER INVALID_NETID BIND <netid> Network not found

TODO: more errors

### Standard network attributes

* `name`: the human-readable name for the network.
* `state`: one of `connected`, `connecting` or `disconnected`. Indicates the
  current state of the connection to the upstream network.

TODO: more attributes

### Examples

Binding to a network:

    C: CAP LS 302
    C: NICK emersion
    C: USER emersion 0 0 :Simon
    S: CAP * LS :sasl=PLAIN bouncer-networks
    C: CAP REQ :sasl bouncer-networks
    [SASL authentication]
    C: BOUNCER BIND 42
    C: CAP END

Listing networks:

    C: BOUNCER LISTNETWORKS
    S: BOUNCER LISTNETWORKS 42 name=Freenode;state=connected
    S: BOUNCER LISTNETWORKS 43 name=My\sAwesome\sNetwork;state=disconnected
    S: BOUNCER LISTNETWORKS *
