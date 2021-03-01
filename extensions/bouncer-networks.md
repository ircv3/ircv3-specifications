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
format. Clients MUST ignore unknown attributes.

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

#### `ADDNETWORK` subcommand

The `ADDNETWORK` subcommand registers a new upstream network in the bouncer.

    BOUNCER ADDNETWORK <attributes>

The bouncer MAY reject this new network for any reason, in this case it MUST
reply with an error. If the request is accepted, the bouncer MUST generate a
new unique network ID. The bouncer MAY populate unspecified attributes with
implementation-defined defaults.

Clients MUST specify at least the `host` attribute.

If the client doesn't specify the `tls` attribute, the server SHOULD use the
default `1`. If the client doesn't specify the `port` attribute, the server
SHOULD use the default `6697` if `tls=1` or `6667` if `tls=0`.

On success, the server replies with:

    BOUNCER ADDNETWORK <netid>

#### `CHANGENETWORK` subcommand

The `CHANGENETWORK` subcommand changes attributes of an existing upstream
network.

    BOUNCER CHANGENETWORK <netid> <attributes>

The bouncer MAY reject the change for any reason, in this case it MUST reply
with an error. At least one attribute MUST be specified by the client.

On success, the server replies with:

    BOUNCER CHANGENETWORK <netid>

#### `DELNETWORK` subcommand

The `DELNETWORK` subcommand removes an existing upstream network.

    BOUNCER DELNETWORK <netid>

The bouncer MAY reject the change for any reason, in this case it MUST reply
with an error.

On success, the server replies with:

    BOUNCER DELNETWORK <netid>

### Network notifications

When a network attributes are updated, the bouncer MUST broadcast a
`BOUNCER NETWORK` message to all connected clients with the updated attributes:

    BOUNCER NETWORK <netid> <attributes>

When a network is removed, the bouncer MUST broadcast a `BOUNCER NETWORK`
message to all connected clients:

    BOUNCER NETWORK <netid> *

### Errors

Errors are returned using the standard replies syntax. The general syntax is:

    FAIL BOUNCER <code> <subcommand> [context...] <description>

If a client sends an unknown subcommand, the server MUST reply with:

    FAIL BOUNCER UNKNOWN_COMMAND <subcommand> :Unknown subcommand

#### `ACCOUNT_REQUIRED` error

If a client sends a `BIND` subcommand before authentication, the server MAY
reply with:

    FAIL BOUNCER ACCOUNT_REQUIRED BIND :Authentication required

#### `REGISTRATION_IS_COMPLETED` error

If a client sends a `BIND` subcommand after registration, the server MAY reply
with:

    FAIL BOUNCER REGISTRATION_IS_COMPLETED BIND :Cannot bind to a network after registration

#### `INVALID_NETID` error

If a client sends a subcommand with an invalid network ID, the server MUST
reply with:

    FAIL BOUNCER INVALID_NETID <subcommand> <netid> :Network not found

#### `INVALID_ATTRIBUTE` error

If a client sends an `ADDNETWORK` or a `CHANGENETWORK` subcommand with an
invalid attribute, the server MUST reply with:

    FAIL BOUNCER INVALID_ATTRIBUTE <subcommand> <netid> <attribute> :Invalid attribute value

If the `subcommand` is `ADDNETWORK`, `netid` MUST be set to the special `*`
value.

#### `READ_ONLY_ATTRIBUTE` error

If a client attempts to change a read-only network attribute using the
`ADDNETWORK` or `CHANGENETWORK` subcommand, the server MUST reply with:

    FAIL BOUNCER READ_ONLY_ATTRIBUTE <subcommand> <netid> <attribute> :Read-only attribute

If the `subcommand` is `ADDNETWORK`, `netid` MUST be set to the special `*`
value.

#### `UNKNOWN_ATTRIBUTE` error

If a client sends an `ADDNETWORK` or a `CHANGENETWORK` subcommand with an
unknown attribute, the server MUST reply with:

    FAIL BOUNCER UNKNOWN_ATTRIBUTE <subcommand> <netid> <attribute> :Unknown attribute

If the `subcommand` is `ADDNETWORK`, `netid` MUST be set to the special `*`
value.

#### `NEED_ATTRIBUTE` error

If a client sends an `ADDNETWORK` subcommand without a mandatory attribute, the
server MUST reply with:

    FAIL BOUNCER NEED_ATTRIBUTE ADDNETWORK <attribute> :Missing required attribute

TODO: more errors

### Standard network attributes

Bouncers MUST recognise the following network attributes:

* `name`: the human-readable name for the network.
* `state` (read-only): one of `connected`, `connecting` or `disconnected`.
  Indicates the current state of the connection to the upstream network.
* `host`: the hostname or literal IP address to connect to.
* `port`: the TCP port to connect to.
* `tls`: `1` to use a TLS connection, `0` to use a cleartext connection.
* `nickname`: the nickname to use during registration.
* `username`: the username to use during registration.
* `realname`: the realname to use during registration.

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

Adding a new network:

    C: BOUNCER ADDNETWORK name=OFTC;host=irc.oftc.net
    S: BOUNCER NETWORK 44 status=connecting
    S: BOUNCER ADDNETWORK 44
    S: BOUNCER NETWORK 44 status=connected

Changing an existing network:

    C: BOUNCER CHANGENETWORK 44 realname=Simon
    S: BOUNCER NETWORK 44 realname=Simon
    S: BOUNCER CHANGENETWORK 44

Removing an existing network:

    C: BOUNCER DELNETWORK 44
    S: BOUNCER NETWORK 44 *
    S: BOUNCER DELNETWORK 44
