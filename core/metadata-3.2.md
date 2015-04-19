---
title: IRCv3.2 Metadata
layout: spec
---
# IRCv3.2 Metadata

Copyright (c) 2012 Kiyoshi Aman <kiyoshi.aman@gmail.com>.

Unlimited redistribution and modification is allowed provided that the above
copyright notice and this permission notice remains intact.

## Introduction

It is generally useful to associate metadata with one's IRC presence, e.g. to
make one's homepage or non-IRC contact details more discoverable. There are
several mechanisms for doing this, but they typically rely on the presence of
services and aren't really suitable for transient metadata such as a user's
current location.

This proposal aims to codify one mechanism for working with metadata: metadata
may be configured through a client-to-server event created specifically for
this purpose.

## METADATA

All metadata subcommands will flow through the outgoing `METADATA` verb.

This specification adds the `metadata-notify` capability for notifications.
Please see the 'Metadata Notifications' section for more information.

For the purposes of this specification, 'targets' are entities for which
metadata may be set. Such entities MUST be a valid nickname or channel. As a
convenient shorthand, an asterisk (`*`) MAY be specified to indicate that the
target is the client itself. Invalid targets MUST be responded to with
`ERR_TARGETINVALID`.

Implementations MUST provide a mechanism for controlling the ability to set or
clear metadata on channels. Such mechanisms SHALL respond to unauthorized
attempts with `ERR_KEYNOPERMISSION`.

Implementations MAY provide a mechanism for limiting visibility of certain
metadata to certain users. Such mechanism is implementation-defined;
for instance, it may depend on some permission level or a flag.

### METADATA GET

This command allows lookup of some keys. The format MUST be as follows:

    METADATA <Target> GET key1 key2 ...

Multiple keys may be given.
The response will be either `RPL_KEYVALUE`, `ERR_KEYINVALID`
or `ERR_NOMATCHINGKEYS` for every key in order.

Servers MAY replace certain metadata, which is considered not visible for the
requesting user, with `ERR_NOMATCHINGKEYS` or with `ERR_KEYNOPERMISSION`.

*Errors*: `ERR_NOMATCHINGKEYS`, `ERR_KEYINVALID`, `ERR_KEYNOPERMISSION`.

### METADATA LIST

The format MUST be as follows:

    METADATA <Target> LIST

This subcommand MUST list all currently-set metadata keys along with their
values. The response will be zero or more `RPL_KEYVALUE` events,
	following by `RPL_METADATAEND` event.

Servers MAY omit certain metadata, which is considered not visible for
the requesting user, or replace it with `ERR_KEYNOPERMISSION`.

In case of invalid target `RPL_METADATAEND` MUST be not sent.

*Errors*: `ERR_NOMATCHINGKEYS`, `ERR_KEYNOPERMISSION`.

### METADATA SET

This subcommand MUST set a required key to an optional value. If no value is
given, the key is removed; otherwise, the value is assigned to the key. The
response MUST be one `RPL_KEYVALUE` event, representing what was actually
stored by the server, and one `RPL_METADATAEND` event. The format of
 `METADATA SET` MUST be as follows:

    METADATA <Target> SET <Key> [:Value]

It is an error for users to set keys on targets for which they lack
authorization from the server, and the server MUST respond with
`ERR_KEYNOPERMISSION`.

*Errors*: `ERR_METADATALIMIT`, `ERR_KEYINVALID`, `ERR_KEYNOTSET`, `ERR_KEYNOPERMISSION`

### METADATA CLEAR

This subcommand MUST remove all metadata, equivalently to using `METADATA SET`
on all currently-set keys with an empty value. The format of `METADATA CLEAR`
MUST be as follows:

    METADATA <Target> CLEAR

The server MUST respond with one `RPL_KEYVALUE` event per cleared key and one
`RPL_METADATAEND` event. It is an error for users to use this subcommand on
targets for which they lack authorization from the server. Servers MAY reject
this subcommand for channels, using `ERR_KEYNOPERMISSION` with an asterisk
(`*`) in the `<Key>` field.

## Metadata Notifications

Metadata notifications are enabled by requesting the `metadata-notify`
capability during capability negotiation. When negotiated, this capability
extends `MONITOR` behaviour to include subscribing users to notifications for
those users they are currently monitoring. Clients are also subscribed to
notifications for channels they join. Clients may discontinue notifications
for users by issuing a disabling `MONITOR` command, and for channels by
parting the channel. Clients are automatically subscribed to notifications for
their own metadata, excluding changes made by the clients themselves.

Notifications use the `METADATA` event, the format of which is as follows:

    METADATA <Target> <Key> <Visibility> :<Value>

`<Target>` refers to the entity which had its metadata changed. `<Visibility>`
MUST be `*` for keys visible to everyone, or a token which describes the
key's visibility status in an implementation-defined way; for instance, it may
be a permission level or flag.

Clients MUST handle all metadata notifications, whether they explicitly
requested them or not.

Metadata propagates to clients automatically under certain conditions:

1. Upon authentication and successful negotiation of `metadata-notify`, clients
   MUST have their non-transient metadata propagated to them. If none exists,
   servers MUST send `ERR_METADATAEND`.
2. Clients SHOULD have current metadata propagated to them.
3. Clients who enable `metadata-notify` after issuing `MONITOR` commands to
   subscribe to users SHOULD have current metadata propagated to them for those
   users.

## Metadata Restrictions

Keys MUST be restricted to the ranges `A-Z`, `a-z`, `0-9`, and `_.:`, and are
case-insensitive. Values are unrestricted, except that they MUST be UTF-8.
Server and client authors cannot assume that keys will carry specific
formatting or meaning.

If a limit is set for keys, it MUST only apply to user-set keys and MUST be
communicated to the user via `RPL_ISUPPORT` as a numeric value on the
`METADATA` key.

## WHOIS

A subset of metadata MAY be sent via the `RPL_WHOISKEYVALUE` event; this
subset SHALL be set explicitly, rather than informatively or as a side-effect
of other events. For a complete view of user metadata, see `METADATA LIST`.

## IRC Daemons

IRC servers may choose to accept or deny any key for any reason, and SHOULD
implement a blacklist or whitelist functionality for this purpose, configurable
by the server operators.

If `METADATA` is supported, it MUST be specified in `RPL_ISUPPORT` using the
`METADATA` key. Servers MAY specify a limit on the number of explicitly-set
keys per-user; the format in that case MUST be `METADATA=<integer>`, where
`<integer>` is the limit.

## Numerics

The numerics 760 through 769 MUST be reserved for metadata, carrying the
following labels and formats:

| No. | Label                 | Format                                   |
| --- | --------------------- | ---------------------------------------- |
| 760 | `RPL_WHOISKEYVALUE`   | `<Target> <Key> <Visibility> :<Value>`   |
| 761 | `RPL_KEYVALUE`        | `<Target> <Key> <Visibility>[ :<Value>]` |
| 762 | `RPL_METADATAEND`     | `:end of metadata`                       |
| 764 | `ERR_METADATALIMIT`   | `<Target> :metadata limit reached`       |
| 765 | `ERR_TARGETINVALID`   | `<Target> :invalid metadata target`      |
| 766 | `ERR_NOMATCHINGKEYS`  | `<Key> :no matching keys`                |
| 767 | `ERR_KEYINVALID`      | `<Key> :invalid metadata key`            |
| 768 | `ERR_KEYNOTSET`       | `<Target> <Key> :key not set`            |
| 769 | `ERR_KEYNOPERMISSION` | `<Target> <Key> :permission denied`      |

The `<Visibility>` field for numerics follows the same rules and requirements
as specified for notifications' visibility.

## Examples

Setting metadata for self:

    METADATA * SET url :http://www.example.com
    :irc.example.com 761 * url * :http://www.example.com
    :irc.example.com 762 :end of metadata

Setting metadata for channel:

    METADATA #example SET url :http://www.example.com
    :irc.example.com 761 #example url * :http://www.example.com
    :irc.example.com 762 :end of metadata

Setting metadata for another user, no permission:

    METADATA user1 SET url :http://www.example.com
    :irc.example.com 769 user1 url :permission denied

Setting metadata for self, limit reached:

    METADATA * SET url :http://www.example.com
    :irc.example.com 764 * :metadata limit reached

Setting metadata for an invalid target:

    METADATA $a:user SET url :http://www.example.com
    :irc.example.com 765 $a:user :invalid metadata target

Setting metadata with an invalid key:

    METADATA user1 SET $url$ :http://www.example.com
    :irc.example.com 767 $url$ :invalid metadata key

Listing metadata, with an implementation-defined visibility field:

    METADATA user1 LIST
    :irc.example.com 761 user1 url * :http://www.example.com
    :irc.example.com 761 user1 im.xmpp * :user1@xmpp.example.com
    :irc.example.com 761 user1 bot-likeliness-score visible-only-for-admin :42
    :irc.example.com 762 :end of metadata

Getting several keys of metadata of the same user:

    METADATA user1 GET blargh splot im.xmpp
    :irc.example.com 766 user1 blargh :no matching keys
    :irc.example.com 766 user1 splot :no matching keys
    :irc.example.com 761 user1 im.xmpp * :user1@xmpp.example.com

User sets metadata on a channel:

    :user1!~user@somewhere.example.com METADATA #example url * :http://www.example.com

External server updates metadata on a channel:

    :irc.example.com METADATA #example url * :http://wiki.example.com

External server sets metadata on a user:

    :irc.example.com METADATA user1 account * :user1
