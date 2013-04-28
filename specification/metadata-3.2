# Metadata Specification

Copyright (c) 2012 Kiyoshi Aman <kiyoshi.aman@gmail.com>.

Unlimited redistribution and modification is allowed provided that the above
copyright notice and this permission notice remains intact.

## Introduction

It is generally useful to associate metadata with one's IRC presence, e.g. to
make one's homepage or non-IRC contact details more discoverable. There are
several mechanisms for doing this, but they typically rely on the presence of
services and aren't really suitable for transient metadata such as a user's
current location.

This proposal aims to codify two coexisting mechanisms for working with
metadata: first, persisting metadata may be configured through services via
NickServ or an appropriate user authentication agent; second, transient
metadata may be configured through a client --> server event created
specifically for this purpose. Both mechanisms will behave identically, save
that services-driven metadata will be encapsulated via `PRIVMSG`.

## METADATA

All metadata subcommands will flow through the outgoing `METADATA` verb.
Metadata may apply to channels, as well; in that case, an optional
argument is provided prior to the subcommand, as outlined in the format for
each, if supported.

This specification adds the `metadata-notify` capability for notifications.
Please see the 'Metadata Notifications' section for more information.

Targets specified by the following subcommands MUST be an asterisk (`*`) when
used for a user's own metadata, a valid nickname, or a valid mask as determined
by the IRC server. Invalid targets MUST be responded to with
`ERR_TARGETINVALID`.

### METADATA LIST

This subcommand MUST list all currently-set metadata keys along with their
values. An optional string may be given to reduce the list to matching keys.
Multiple keys may be given, separated by spaces. The response will be one or
more `RPL_KEYVALUE` events and a closing `RPL_METADATAEND` event. The format
of `METADATA LIST` MUST be as follows:

`METADATA <Target> LIST [:String]`

*Errors*: ERR_NOMATCHINGKEYS

### METADATA SET

This subcommand MUST set a required key to an optional value. If no value is
given, the key is removed; otherwise, the value is assigned to the key. The
response MUST be one `RPL_KEYVALUE` event and one `RPL_METADATAEND` event. The
format of `METADATA SET` MUST be as follows:

`METADATA <Target> SET <Key> [:Value]`

It is an error for users to set keys for other users, for services accounts
other than their own (without an appropriate operator privilege), for channels
for which they lack implementation-defined permission, or for
implementation-defined keys which would require special access which the user
does not possess.

*Errors*: ERR_KEYINVALID, ERR_KEYNOTSET, ERR_KEYNOPERMISSION

### METADATA CLEAR

This subcommand MUST remove all metadata, equivalently to using METADATA SET
on all keys with an empty value. The format of `METADATA CLEAR` MUST be as
follows:

`METADATA <Target> CLEAR`

This subcommand has the same conditions and errors as the `METADATA SET`
subcommand.

## Metadata Notifications

Notifications, e.g. from metadata subscriptions, MUST be passed to the client
using the incoming METADATA verb, the format of which is as follows:

`METADATA <Source> <Target> <Key> :<Value>`

`<Source>` identifies the source of the change. `<Target>` refers to the target
specified by the `MONITOR` command.

This specification extends `MONITOR` to enable metadata notifications using the
above verb. Clients MUST handle all `METADATA` verbs, regardless of
subscription status. Notifications MUST be sent for the client's own keys,
regardless of subscription status.

## Metadata Restrictions

Keys MUST be restricted to the ranges A-Z, a-z, and 0-9, and are
case-insensitive; they MUST, moreover, be namespaced using the period (.) as a
separator. Values are unrestricted, except that they MUST be UTF-8; binary data
MUST use the 'data:' URI scheme as standardized by browsers and SHOULD be
discouraged.

### Key Registry

There shall be a key registry, with the intention of standardizing keys for
ease of use among server and client authors. It shall be maintained by the
IRCv3 working group or a suitably-delegated authority. Several namespaces
will be defined initially, as follows:

* The 'server' namespace is intended for keys which the user cannot set, such
  as SSL certificate fingerprints. Some keys MAY have restricted visibility.
* The 'user' namespace is intended for keys which the user can set and which
  carry meaning relevant only, or mostly, to users.
* The 'client' namespace is intended for keys which the user can set and which
  describe the user's client.
* The 'ext' namespace is intended for keys which have not been formally
  registered. Server and client authors are advised that they cannot rely on
  this namespace to carry any standardized meaning. The main namespaces MAY be
  replicated under this namespace, except for 'private'.
* The 'private' namespace is intended for server-internal keys and MUST be seen
  only with the appropriate server operator permissions. Keys in this
  namespace are not required to be registered.

If a limit is set for keys, it MUST only apply to user-set keys and MUST be
communicated to the user via `RPL_ISUPPORT` as a numeric value on the
`METADATA` key.

### Predefined Keys

The following keys are predefined for the purposes of the key registry outlined
above:

| Key            | Meaning                                                      |
| -------------- | ------------------------------------------------------------ |
| server.certfp  | SSL certificate fingerprint                                  |
| user.email     | User's email address                                         |
| user.phone     | User's phone number                                          |
| user.website   | User's website                                               |
| user.im.*      | IM handles; the * is replaced with the relevant service name |
| user.playing   | Music the user is currently listening to                     |
| user.status    | The user's current status                                    |
| client.name    | Client's name                                                |
| client.version | Client version                                               |

Except for `server.certfp`, all of these keys SHOULD be considered to be
free-flowing text with no inherent meaning. `server.certfp` MUST be presented
as hexadecimal.

## WHOIS

Metadata MUST appear in `WHOIS` output using `RPL_WHOISKEYVALUE` if any is defined
for the user.

## Services

Metadata SHOULD be presentable via NickServ or similar service in order to
allow for persistent metadata; the format for encapsulating `METADATA` events
for persistance MUST be as follows:

`PRIVMSG <Service> :<Metadata command>`

where `<Service>` is the relevant service and `<Metadata command>` is any 
`METADATA` subcommand as outlined previously. Responses MUST be presented
appropriately to the service rather than through RPL_* or ERR_* events.

In addition, services MAY display metadata via the `INFO` command with the
relevant service and account name.

## IRC Daemons

IRCds MAY have a blacklist or whitelist and may have an option to enforce
keys against either or neither of them. Implementations may block keys which
might result in impersonation. It is an error for user-set metadata to have
any effect on server operations.

If `METADATA` is supported, it MUST be specified in RPL_ISUPPORT using the
`METADATA` key.

## Numerics

The numerics 760 through 769 MUST be reserved for metadata, carrying the
following labels and formats:

| No. | Label               | Format                              |
| --- | ------------------- | ----------------------------------- |
| 760 | RPL_WHOISKEYVALUE   | `<Target> <Key> :<Value`            |
| 761 | RPL_KEYVALUE        | `<Target> <Key>[ :<Value>]`         |
| 762 | RPL_METADATAEND     | `:end of metadata`                  |
| 765 | ERR_TARGETINVALID   | `<Target> :invalid metadata target` |
| 766 | ERR_NOMATCHINGKEYS  | `<String> :no matching keys`        |
| 767 | ERR_KEYINVALID      | `<Key> :invalid metadata key`       |
| 768 | ERR_KEYNOTSET       | `<Target> <Key> :key not set`       |
| 769 | ERR_KEYNOPERMISSION | `<Target> <Key> :permission denied` |

For `RPL_WHOISKEYVALUE`, the `<Target>` is the usual WHOIS target. For
all other numerics, the `<Target>` is the nick or valid server-defined
mask which triggered the numeric response.
