---
title: Metadata
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Kiyoshi Aman"
    period: "2012"
    email: "kiyoshi.aman@gmail.com"
  -
    name: "Attila Molnar"
    period: "2015-2016"
    email: "attilamolnar@hush.com"
  -
    name: "James Wheare"
    period: "2018"
    email: "james@irccloud.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `metadata` capability name. Instead, implementations SHOULD
use the `draft/metadata` capability name to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

It is generally useful to associate metadata with one's IRC presence, e.g. to
make one's homepage or non-IRC contact details more discoverable. There are
several mechanisms for doing this, but they typically rely on the presence of
services and aren't really suitable for transient metadata such as a user's
current location.

This proposal aims to codify one mechanism for working with metadata: metadata
may be configured through a client-to-server event created specifically for
this purpose.

## Mechanisms

This document defines the following new protocol features:

* Capability: `metadata`
* Server notification: `METADATA`
* Client command: `METADATA`
* Server reply and error numerics

### Capability

The `metadata` capability and value indicates that a server supports metadata, and communicates any configuration value keys that may affect the level of support.

The ABNF format of the `metadata` capability and value is:

    capability ::= 'metadata' ['=' tokens]
    tokens     ::= token [',' token]*
    token      ::= key ['=' value]
    key        ::= <sequence of a-zA-Z0-9_.:- >
    value      ::= <utf8>

The value keys are defined as follows:

* `maxsub`: the maximum number of keys a client is allowed in its subscripion list. See the [Metadata subscriptions](#todo) section for more details.
* `maxkey`: the maximum number of keys a client is allowed to set on its own nickname.


TODO is this still required? include it for back compat?
If `METADATA` is supported, it MUST be specified in `RPL_ISUPPORT` using the
`METADATA` key. Servers MAY specify a limit on the number of explicitly-set
keys per-user; the format in that case MUST be `METADATA=<integer>`, where
`<integer>` is the limit.


### Server notification

Clients that request the `metadata` capability MUST be able to handle the `METADATA` server notification. After negotiating this capability, servers MAY send these notifications to clients at any time.

The format of the `METADATA` server notication is:

    METADATA <Target> <Key> <Visibility> <Value>

`Target` MUST be a valid nickname or channel name.

`Key` MUST be a valid key name

`Visibility` MUST be an asterisk (`*`) for keys visible to everyone, or an implementation-defined value which describes the key's visibility status; for instance, it MAY be a permission level or flag. (TODO should we define a prefix like STATUSMSG for this value?)

`Value` MUST be a UTF-8 encoded value.

### Client command

The format of the `METADATA` client command is:

    METADATA <Target> <Subcommand> [<Param 1> ... [<Param n>]]

`Target` MUST be a valid nickname or channel name. Clients MAY use the asterisk symbol (`*`) when targeting their own nickname.

`Subcommand` MUST be one of the following, described in detail along with any `Param` further in the document:

* [GET](#todo)
* [LIST](#todo)
* [SET](#todo)
* [CLEAR](#todo)
* [SUB](#todo)
* [UNSUB](#todo)
* [SUBS](#todo)
* [SYNC](#todo)

### Numerics

The following numerics 760 through 775 are reserved for metadata, with these labels and parameters:

| No. | Label                     | Parameters                               |
| --- | ------------------------- | ---------------------------------------- |
| 760 | `RPL_WHOISKEYVALUE`       | `<Target> <Key> <Visibility> :<Value>`   |
| 761 | `RPL_KEYVALUE`            | `<Target> <Key> <Visibility>[ :<Value>]` |
| 762 | `RPL_METADATAEND`         | `:end of metadata`                       |
| 764 | `ERR_METADATALIMIT`       | `<Target> :metadata limit reached`       |
| 765 | `ERR_TARGETINVALID`       | `<Target> :invalid metadata target`      |
| 766 | `ERR_NOMATCHINGKEY`       | `<Target> <Key> :no matching key`        |
| 767 | `ERR_KEYINVALID`          | `:<InvalidKey>`                          |
| 768 | `ERR_KEYNOTSET`           | `<Target> <Key> :key not set`            |
| 769 | `ERR_KEYNOPERMISSION`     | `<Target> <Key> :permission denied`      |
| 770 | `RPL_METADATASUBOK`       | `:<Key1> [<Key2> ...]`                   |
| 771 | `RPL_METADATAUNSUBOK`     | `:<Key1> [<Key2> ...]`                   |
| 772 | `RPL_METADATASUBS`        | `:<Key1> [<Key2> ...]`                   |
| 773 | `ERR_METADATATOOMANYSUBS` | `<Key>`                                  |
| 774 | `ERR_METADATASYNCLATER`   | `<Target> [<RetryAfter>]`                |
| 775 | `ERR_METADATARATELIMIT`   | `<Target> <Key> <RetryAfter> :<Value>`   |

Reference table of numerics and the subcommands of `METADATA` or any other commands that produce them:

| Label                     | GET | LIST | SET | CLEAR | SUB | UNSUB | SUBS | SYNC | Other   |
| ------------------------- | --- | ---- | --- | ----- | --- | ----- | ---- | ---- | ------- |
| `RPL_WHOISKEYVALUE`       |     |      |     |       |     |       |      |      | `WHOIS` |
| `RPL_KEYVALUE`            | *   | *    | *   | *     |     |       |      |      |         |
| `RPL_METADATAEND`         |     | *    | *   | *     | *   | *     | *    |      |         |
| `ERR_METADATALIMIT`       |     |      | *   |       |     |       |      |      |         |
| `ERR_TARGETINVALID`       | *   | *    | *   | *     | *   | *     | *    | *    |         |
| `ERR_NOMATCHINGKEY`       | *   |      |     |       |     |       |      |      |         |
| `ERR_KEYINVALID`          | *   |      | *   |       | *   | *     |      |      |         |
| `ERR_KEYNOTSET`           |     |      | *   |       |     |       |      |      |         |
| `ERR_KEYNOPERMISSION`     | *   | *    | *   |       | *   | *     |      |      |         |
| `RPL_METADATASUBOK`       |     |      |     |       | *   |       |      |      |         |
| `RPL_METADATAUNSUBOK`     |     |      |     |       |     | *     |      |      |         |
| `RPL_METADATASUBS`        |     |      |     |       |     |       | *    |      |         |
| `ERR_METADATATOOMANYSUBS` |     |      |     |       | *   |       |      |      |         |
| `ERR_METADATASYNCLATER`   |     |      |     |       |     |       |      | *    | `JOIN`  |
| `ERR_METADATARATELIMIT`   |     |      | *   |       |     |       |      |      |         |

Each subcommand documentation describes the reply and error numerics it expects from the server, but here are brief descriptions of numerics that are used for multiple commands:

Replies:

* `RPL_KEYVALUE` reports the values of metadata keys. The `Visibility` parameter is defined as specified in the [server notification](#todo) format.
* `RPL_METADATAEND` delimits the end of a sequence of metadata replies.

Errors:

* `ERR_TARGETINVALID` when a client refers to an invalid target.
* `ERR_KEYINVALID` when a client refers to an invalid key.
* `ERR_KEYNOPERMISSION` when a client attempts to access or set a key on a target without sufficient permission.

### Keys and values

Key names MUST be restricted to the ranges `A-Z`, `a-z`, `0-9`, and `_.:-` and are case-insensitive. Key names MUST not start with a colon (`:`).

Values are unrestricted, except that they MUST be encoded using UTF-8.

The expected client behaviour of individual metadata keys SHOULD be defined in separate specifications and listed in the IRCv3 extension registry.

Servers MAY impose a limit on the number of keys a client is allowed to set via the `maxkey` capability value.

Servers MAY impose a limit on the number of keys a client is allowed in its subscripion list via the `maxsub` capability value.

## Subcommands

### METADATA GET

This command allows lookup of some keys. The format MUST be as follows:

    METADATA <Target> GET key1 key2 ...

Multiple keys may be given.
The response will be either `RPL_KEYVALUE`, `ERR_KEYINVALID`
or `ERR_NOMATCHINGKEY` for every key in order.

Servers MAY replace certain metadata, which is considered not visible for the
requesting user, with `ERR_NOMATCHINGKEY` or with `ERR_KEYNOPERMISSION`.

*Errors*: `ERR_NOMATCHINGKEY`, `ERR_KEYINVALID`, `ERR_KEYNOPERMISSION`.

### METADATA LIST

The format MUST be as follows:

    METADATA <Target> LIST

This subcommand MUST list all currently-set metadata keys along with their
values. The response will be zero or more `RPL_KEYVALUE` events,
followed by a `RPL_METADATAEND` event.

Servers MAY omit certain metadata, which is considered not visible for
the requesting user, or replace it with `ERR_KEYNOPERMISSION`.

In case of invalid target `RPL_METADATAEND` MUST NOT be sent.

*Errors*: `ERR_KEYNOPERMISSION`.

### METADATA SET

This subcommand is used to set a required key to an optional value. If no value
is given, the key is removed; otherwise, the value is assigned to the key. The
format of `METADATA SET` MUST be as follows:

    METADATA <Target> SET <Key> [:Value]

Servers MUST respond to requests to set or remove a key whose name is invalid
with only an `ERR_KEYINVALID` event and fail the request.

Servers MUST respond to requests to remove a key that has a valid name but is
not set with only an `ERR_KEYNOTSET` event and fail the request.

Servers MAY respond to certain keys, considered not settable by the requesting
user, or otherwise disallowed by the server, with `ERR_KEYNOPERMISSION`.

It is an error for users to set keys on targets for which they lack
authorization from the server, and the server MUST respond with
`ERR_KEYNOPERMISSION` and fail the request.

Servers MAY respond with only an `ERR_METADATARATELIMIT` event and fail the
request. When a client receives an `ERR_METADATARATELIMIT` event, it SHOULD
retry the `METADATA SET` request at a later time. If the
`ERR_METADATARATELIMIT` event contains the OPTIONAL `<RetryAfter>` parameter,
the parameter value MUST be a positive integer indicating the minimum number
of seconds the client SHOULD wait before retrying the request.

If the request is successful the server MUST carry out the requested change and
the response MUST be one `RPL_KEYVALUE` event, representing what was actually
stored by the server, and one `RPL_METADATAEND` event.

*Errors*: `ERR_METADATALIMIT`, `ERR_KEYINVALID`, `ERR_KEYNOTSET`, `ERR_KEYNOPERMISSION`, `ERR_METADATARATELIMIT`

### METADATA CLEAR

This subcommand MUST remove all metadata, equivalently to using `METADATA SET`
on all currently-set keys with an empty value. The format of `METADATA CLEAR`
MUST be as follows:

    METADATA <Target> CLEAR

The server MUST respond with one `RPL_KEYVALUE` event per cleared key and one
`RPL_METADATAEND` event.

Servers MAY omit certain metadata, which is considered not settable by
the requesting user, or replace it with `ERR_KEYNOPERMISSION`.

It is an error for users to use this subcommand on targets for which they lack
authorization from the server. Servers MAY reject this subcommand for channels,
using `ERR_KEYNOPERMISSION` with an asterisk (`*`) in the `<Key>` field.

*Errors*: `ERR_KEYNOPERMISSION`

### METADATA SUB

This subcommand is used to subscribe to metadata keys.

Syntax:

    METADATA * SUB <key1> [<key2> ...]

The server MUST reply with zero or more numerics of the following types in any
order: `RPL_METADATASUBOK`, `ERR_KEYINVALID`, `ERR_KEYNOPERMISSION` and zero or
one `ERR_METADATATOOMANYSUBS` numeric.
The server MUST end the reply with one `RPL_METADATAEND` numeric.

The server MUST process the keys in the given order. This is critical when
determining which keys the client gets subscribed to in case the server limits
the number of keys the client can subscribe to.

If the client is subscribed to too many keys then the server MUST include a
`ERR_METADATATOOMANYSUBS` numeric in its reply and not process any further keys
in the command.

The `<key>` parameter of this numeric is the first key that the client was not
subscribed to.

If the client successfully subscribes to a key it is not subscribed to then the
key MUST appear in a `RPL_METADATASUBOK` reply numeric.

If the client tries to subscribe to a key it is already subscribed to then the
client remains subscribed to the key. In case such a key is processed in a
request the key MUST appear at least once in a `RPL_METADATASUBOK` numeric in
the reply.

Servers MUST respond to requests to subscribe to a key whose name is invalid
with a `ERR_KEYINVALID` numeric.

Servers MAY additionally respond to requests to subscribe to a key that the
client has no privilege to access with a `ERR_KEYNOPERMISSION` numeric.
Even if a server does that, the subscription MUST still be successful.
This is because in this case the `ERR_KEYNOPERMISSION` numeric only serves as a
warning, indicating that the client will not receive METADATA events about this
key unless it acquires the necessary (implementation defined) privileges later.

### METADATA UNSUB

This subcommand is used to unsubscribe from metadata keys.

Syntax:

    METADATA * UNSUB <key1> [<key2> ...]

The reply of the server MUST be zero or more numerics of the following types
in any order: `RPL_METADATAUNSUBOK`, `ERR_KEYINVALID`.
Then the server MUST end the reply with one `RPL_METADATAEND` numeric.

If a client successfully unsubscribes from a key it is subscribed to then the
key MUST appear in a `RPL_METADATAUNSUBOK` reply numeric.

If a client tries to unsubscribe from a key that it is not subscribed to then
the client remains not subscribed to the key and the key MUST appear at least
once in a `RPL_METADATAUNSUBOK` numeric in the reply.

Servers MUST respond to requests to subscribe to a key whose name is invalid
with a `ERR_KEYINVALID` numeric.

### METADATA SUBS

This subcommand can be used to get a list of keys which the client is
subscribed to.

Syntax:

    METADATA * SUBS

The server MUST reply with zero or more `RPL_METADATASUBS` numerics and then
one `RPL_METADATAEND` numeric.

The replied `RPL_METADATASUBS` numerics, collectively, MUST contain all keys
the client is subscribed to exactly once and MUST NOT contain keys the client
is not subscribed to.

The order of the keys is undefined.

### METADATA SYNC

This subcommand requests the full synchronization of metadata associated with
the given target.

Syntax:

    METADATA <Target> SYNC

The result of this subcommand is either zero or more METADATA events on
success, or a `ERR_METADATASYNCLATER` numeric if the synchronization cannot be
performed at this time.

For details, please see the Postponed synchronization subsection of the
Metadata notifications section.

## Notification Mechanics (TODO merge with Metadata Notifications)

A client can either be subscribed to a key, or not subscribed to it.

The server MUST allow a client to subscribe to any valid keys, even to
privileged keys when the client has no privilege to access that key at
the time of subscription.

If a client is subscribed to a metadata key and has adequate privileges to get
notifications about that key then it gets METADATA events about the key as
described [here](http://ircv3.net/specs/core/metadata-3.2.html#metadata-notifications).

If a client is not subscribed to a metadata key then it will not get METADATA
events about it, however the client can use `METADATA GET`, `METADATA LIST` or
other means available to obtain the value of the key.

By default, the client is not subscribed to any keys.

Managing subscriptions are possible with the protocol described below.

Clients MUST handle the case where a metadata notification is sent even if they
haven't subscribed to any key.

## Compatibility with `metadata-notify`

It is pointless to use the new metadata notify cap described in this document
and `metadata-notify` (as described in the [metadata v3.2 specification](http://ircv3.net/specs/core/metadata-3.2.html))
together.

This is because `metadata-notify` implicitly subscribes the client to all keys,
while metadata notify v2 requires the client to tell the keys it wants to
subscribe to.

Servers and clients MAY support both of the mentioned extensions, but MUST NOT
negotiate both of them at the same time in the same connection.

If both are available, the new metadata notify cap SHOULD be used.

## Metadata Notifications

Metadata notifications are enabled by requesting the OPTIONAL `metadata-notify`
capability during capability negotiation. When negotiated, this capability
extends `MONITOR` behaviour to include subscribing users to notifications for
those users they are currently monitoring. Clients are also subscribed to
notifications for channels they join. Clients may discontinue notifications
for users by issuing a disabling `MONITOR` command, and for channels by
parting the channel. Clients are automatically subscribed to notifications for
their own metadata, excluding changes made by the clients themselves.

Client subscriptions to a channel includes notifications for other users in
the channel, regardless of `MONITOR` state.

Notifications use the `METADATA` event, the format of which is as follows:

    METADATA <Target> <Key> <Visibility>[ :<Value>]

`<Target>` refers to the entity which had its metadata changed. `<Visibility>`
MUST be `*` for keys visible to everyone, or a token which describes the
key's visibility status in an implementation-defined way; for instance, it may
be a permission level or flag.

Clients MUST handle all metadata notifications, whether they explicitly
requested them or not.

Metadata propagates to clients automatically under certain conditions:

1. Upon authentication and successful negotiation of `metadata-notify`, clients
   MUST have their non-transient metadata propagated to them. If none exists,
   servers MUST send `RPL_METADATAEND`.
2. Clients SHOULD have current metadata of targets propagated to them upon
   subscription.
3. Clients who enable `metadata-notify` after issuing `MONITOR` commands to
   subscribe to users SHOULD have current metadata propagated to them for those
   users.

### Postponed synchronization

It can happen that a server needs to send a large number of `METADATA` events
to a client due to the client subscribing to many targets at once.
For example, this can happen if the client joins a large channel or when the
client is already on some channels and turns on the `metadata`
capability. In this case the server MAY choose to not propagate the metadata
of the newly subscribed targets to the client when the join or when the
`CAP REQ` happens, but send a `ERR_METADATASYNCLATER` numeric instead.

#### Handling `ERR_METADATASYNCLATER`

`ERR_METADATASYNCLATER` signals that the target specified in the numeric has
some metadata set that the client SHOULD request synchronization of at a later
time.

The client can then use the `METADATA SYNC` subcommand to request the
synchronization of the metadata of the target.

The `[<RetryAfter>]` parameter, if present, indicates the number of seconds
the client SHOULD wait before sending the `METADATA SYNC` request for the
`<Target>`.

## WHOIS

A subset of metadata MAY be sent via the `RPL_WHOISKEYVALUE` event; this
subset SHALL be set explicitly, rather than informatively or as a side-effect
of other events. For a complete view of user metadata, see `METADATA LIST`.

## Examples

Setting metadata for self:

    METADATA * SET url :http://www.example.com
    :irc.example.com RPL_KEYVALUE * url * :http://www.example.com
    :irc.example.com RPL_METADATAEND :end of metadata

Setting metadata for channel:

    METADATA #example SET url :http://www.example.com
    :irc.example.com RPL_KEYVALUE #example url * :http://www.example.com
    :irc.example.com RPL_METADATAEND :end of metadata

Setting metadata for another user, no permission:

    METADATA user1 SET url :http://www.example.com
    :irc.example.com ERR_KEYNOPERMISSION user1 url :permission denied

Setting metadata for self, limit reached:

    METADATA * SET url :http://www.example.com
    :irc.example.com ERR_METADATALIMIT * :metadata limit reached

Setting metadata for an invalid target:

    METADATA $a:user SET url :http://www.example.com
    :irc.example.com ERR_TARGETINVALID $a:user :invalid metadata target

Setting metadata with an invalid key:

    METADATA user1 SET $url$ :http://www.example.com
    :irc.example.com ERR_KEYINVALID $url$

Listing metadata, with an implementation-defined visibility field:

    METADATA user1 LIST
    :irc.example.com RPL_KEYVALUE user1 url * :http://www.example.com
    :irc.example.com RPL_KEYVALUE user1 im.xmpp * :user1@xmpp.example.com
    :irc.example.com RPL_KEYVALUE user1 bot-likeliness-score visible-only-for-admin :42
    :irc.example.com RPL_METADATAEND :end of metadata

Getting several keys of metadata of the same user:

    METADATA user1 GET blargh splot im.xmpp
    :irc.example.com ERR_NOMATCHINGKEY user1 blargh :no matching key
    :irc.example.com ERR_NOMATCHINGKEY user1 splot :no matching key
    :irc.example.com RPL_KEYVALUE user1 im.xmpp * :user1@xmpp.example.com

User sets metadata on a channel:

    :user1!~user@somewhere.example.com METADATA #example url * :http://www.example.com

External server updates metadata on a channel:

    :irc.example.com METADATA #example url * :http://wiki.example.com

External server sets metadata on a user:

    :irc.example.com METADATA user1 account * :user1

Server rate limits setting metadata with a RetryAfter value

    METADATA * SET url :http://www.example.com
    :irc.example.com ERR_METADATARATELIMIT * url 5 :http://www.example.com

Server rate limits setting metadata with no RetryAfter value

    METADATA * SET url :http://www.example.com
    :irc.example.com ERR_METADATARATELIMIT * url * :http://www.example.com

Client joins a channel, gets `ERR_METADATASYNCLATER` and requests a sync later

    C: JOIN #bigchan
    S: modernclient!modernclient@example.com JOIN #bigchan
    S: :irc.example.com 353 modernclient @ #bigchan :user1 user2 user3 user4 user5 ...
    S: :irc.example.com 353 modernclient @ #bigchan :user51 user52 user53 user54 ...
    S: :irc.example.com 353 modernclient @ #bigchan :user101 user102 user103 user104 ...
    S: :irc.example.com 353 modernclient @ #bigchan :user151 user152 user153 user154 ...
    S: :irc.example.com 366 modernclient #bigchan :End of /NAMES list.
    S: :irc.example.com ERR_METADATASYNCLATER modernclient #bigchan 4
    
    client waits 4 seconds
    
    C: METADATA #bigchan SYNC
    S: :irc.example.com ERR_METADATASYNCLATER modernclient #bigchan 6
    
    client waits 6 seconds
    
    C: METADATA #bigchan SYNC
    S: :irc.example.com METADATA user52 foo * :example value 1
    S: :irc.example.com METADATA user2 bar * :second example value 
    S: :irc.example.com METADATA user1 foo * :third example value
    S: :irc.example.com METADATA user1 bar * :this is another example value
    S: :irc.example.com METADATA user152 baz * :Lorem ipsum
    S: :irc.example.com METADATA user3 website * :www.example.com
    S: :irc.example.com METADATA user152 bar * :dolor sit amet

    ...and many more metadata messages   


## More Examples (TODO merge with previous)

These examples show the labels of the numerics (e.g. `RPL_METADATASUBOK`)
instead of their number (e.g. `775`) in order to aid understanding.
In a real implementation, the messages always contain the number of numerics,
not a label.

All examples begin with the client not being subscribed to any keys.

### Basic subscriping and unsubscribing

    C: METADATA * SUB avatar website foo bar
    S: :irc.example.com RPL_METADATASUBOK modernclient :avatar website foo bar
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * UNSUB foo bar
    S: :irc.example.com RPL_METADATAUNSUBOK modernclient :bar foo
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Multiple `RPL_METADATASUBOK` numerics in reply to `METADATA SUB`

    C: METADATA * SUB avatar website foo bar baz
    S: :irc.example.com RPL_METADATASUBOK modernclient :avatar website
    S: :irc.example.com RPL_METADATASUBOK modernclient :foo
    S: :irc.example.com RPL_METADATASUBOK modernclient :bar baz
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Invalid key name in reply to subscription

    C: METADATA * SUB foo $url bar
    S: :irc.example.com RPL_METADATASUBOK modernclient :foo bar
    S: :irc.example.com ERR_KEYINVALID modernclient $url :invalid metadata key
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### "Subscribed to too many keys" error in reply to subscription 1

The client first successfully subscribes to some keys and later it tries to
subscribe to some more keys, unsuccessfully.

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com RPL_METADATASUBOK modernclient :website avatar foo bar baz
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUB email city
    S: :irc.example.com ERR_METADATATOOMANYSUBS modernclient email
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :website avatar foo bar baz
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### "Subscribed to too many keys" error in reply to subscription 2

This is like the previous case, except when the second METADATA SUB happens
the server accepts the first 2 keys (`email`, `city`) but not the rest
(`country`, `bar`, `baz`).

    C: METADATA * SUB website avatar foo
    S: :irc.example.com RPL_METADATASUBOK modernclient :website avatar foo
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUB email city country bar baz
    S: :irc.example.com ERR_METADATATOOMANYSUBS modernclient country
    S: :irc.example.com RPL_METADATASUBOK modernclient :email city
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :website avatar city foo email
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### "Subscribed to too many keys" error in reply to subscription 3

In this case, the client is trying to subscribe to a key that it is already
subscribed to (`website`), but the key is not processed because the limit
imposed by the server on the number of subscribed keys is reached before the
`website` key is processed by the server. The client, however, successfully
subscribes to the `foo` key which was also in the second request, but it
appeared before the `website` key.

    C: METADATA * SUB avatar website
    S: :irc.example.com RPL_METADATASUBOK modernclient :avatar website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUB foo website avatar
    S: :irc.example.com ERR_METADATATOOMANYSUBS modernclient website
    S: :irc.example.com RPL_METADATASUBOK modernclient :foo
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :avatar foo website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Querying the list of subscribed keys 1

The server replies with a single `RPL_METADATASUBS` numeric.

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com RPL_METADATASUBOK modernclient :website avatar foo bar baz
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :avatar bar baz foo website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Querying the list of subscribed keys 2

The server replies with multiple `RPL_METADATASUBS` numerics.

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com RPL_METADATASUBOK modernclient :website avatar foo bar baz
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :avatar
    S: :irc.example.com RPL_METADATASUBS modernclient :bar baz
    S: :irc.example.com RPL_METADATASUBS modernclient :foo website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Empty list of subscribed keys

In this case, there are no `RPL_METADATASUB` numerics sent.

    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Unsubscribing

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com RPL_METADATASUBOK modernclient :website avatar foo bar baz
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :avatar bar baz foo website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * UNSUB bar foo baz
    S: :irc.example.com RPL_METADATAUNSUBOK modernclient :baz foo bar
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :avatar website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Subscribing to the same key multiple times 1

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com RPL_METADATASUBOK modernclient :website avatar foo bar baz
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :avatar bar baz foo website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUB avatar website
    S: :irc.example.com RPL_METADATASUBOK modernclient :avatar website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :avatar bar baz foo website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Subscribing to the same key multiple times 2

The client (erroneously) subscribes to the same key twice in the same command.
The server is free to include the key being subscribed to in the
`RPL_METADATASUBOK` numeric once or twice.

In both cases, the key will only appear once in the reply to a following
`METADATA SUBS` command.

Once:

    C: METADATA * SUB avatar avatar
    S: :irc.example.com RPL_METADATASUBOK modernclient :avatar
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :avatar
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

Twice:

    C: METADATA * SUB avatar avatar
    S: :irc.example.com RPL_METADATASUBOK modernclient :avatar avatar
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :avatar
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Unsubscribing from a non-subscribed key 1

    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * UNSUB website
    S: :irc.example.com RPL_METADATAUNSUBOK modernclient :website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUB website
    S: :irc.example.com RPL_METADATASUBOK modernclient :website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Unsubscribing from a non-subscribed key 2

The client (erroneously) unsubscribes from the same key twice in the same
command. The server is free to include the key being unsubscribed from in the
`RPL_METADATAUNSUBOK` numeric once or twice.

Once:

    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * UNSUB website website
    S: :irc.example.com RPL_METADATAUNSUBOK modernclient :website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

Twice:

    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * UNSUB website website
    S: :irc.example.com RPL_METADATAUNSUBOK modernclient :website website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Subscribing to a key which requires privileges but without privileges

    C: METADATA * SUB avatar secretkey website
    S: :irc.example.com ERR_KEYNOPERMISSION modernclient modernclient secretkey :permission denied
    S: :irc.example.com RPL_METADATASUBOK modernclient :secretkey website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :secretkey website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Subscribing to invalid keys and a key which requires privileges but without privileges

    C: METADATA * SUB $invalid1 secretkey1 $invalid2 secretkey2 website
    S: :irc.example.com ERR_KEYNOPERMISSION modernclient modernclient secretkey1 :permission denied
    S: :irc.example.com ERR_KEYINVALID modernclient $invalid1 :invalid metadata key
    S: :irc.example.com ERR_KEYNOPERMISSION modernclient modernclient secretkey2 :permission denied
    S: :irc.example.com ERR_KEYINVALID modernclient $invalid2 :invalid metadata key
    S: :irc.example.com RPL_METADATASUBOK modernclient :secretkey1 secretkey2 website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata
    C: METADATA * SUBS
    S: :irc.example.com RPL_METADATASUBS modernclient :secretkey1 secretkey2 website
    S: :irc.example.com RPL_METADATAEND modernclient :end of metadata

### Capability value in `CAP LS` 1 

    C: CAP LS 302
    S: CAP * LS :userhost-in-names draft/metadata=foo,maxsub=50,bar multi-prefix

### Capability value in `CAP LS` 2

    C: CAP LS 302
    S: CAP * LS :draft/metadata=maxsub=25 multi-prefix invite-notify


### Non-normative

The following examples describe how an implementation might use certain
features. Unlike previous examples, they are in no way intended to guide
implementations' behaviour.

Notification for a user becoming an operator:

    :OperServ!OperServ@services.int METADATA user1 services.operclass oper:auspex :services-root

## Errata

* Earlier version of this spec specified ERR_NOMATCHINGKEY with no `<Target>`.
  This did not match examples and being specific with this numeric was desired.
* Earlier version of this spec specified that the `<Value>` parameter of
  `METADATA` events was required. It was decided `METADATA` events should also
  be able to tell clients about metadata keys that have been removed.
* Earlier versions of this spec were ambiguous about the behavior of
  `METADATA SET`.
* Earlier versions of this spec did not specify `metadata-notify` as OPTIONAL.
* Due to discovered issues around rate-limiting and notifications being solved,
  this specification has been deprecated for the time being.
* Earlier versions of this spec lacked rate limiting protocol mechanics.
* Earlier versions of this spec lacked support for delayed synchronization
  and `METADATA SYNC`.