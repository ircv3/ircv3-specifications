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
  -
    name: "Daniel Oaks"
    period: "2021"
    email: "daniel@danieloaks.net"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `metadata` capability name. Instead, implementations SHOULD
use the `draft/metadata` capability name to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

It is useful to associate metadata with one's IRC presence, e.g. to
make one's homepage or non-IRC contact details more discoverable. There are
several mechanisms for doing this, but they normally rely on the presence of
services and aren't really suitable for transient metadata like a user's
current location.

This feature lays out a command that can be used to set metadata, and a message that can be used to receive metadata updates from the server.

Clients can set metadata on themselves and on channels. Privileged users (for example, network admins) may be able to set certain metadata on other users as well, and set special keys on themselves or channels.

## Notifications

Clients can either be subscribed to a key, or not subscribed to it. By default, clients are not subscribed to any keys. They can subscribe and unsubscribe from keys using the [`SUB`](#metadata-sub) and [`UNSUB`](#metadata-unsub) subcommands. The server MUST allow clients to subscribe to any valid keys, including privileged keys the client cannot access at the time of subscription.

The client will receive [`METADATA` messages](#metadata-server-message) about the keys they are subscribed to. They can also use the [`GET`](#metadata-get) and [`LIST`](#metadata-list) subcommands to receive information about keys they are not subscribed to.

Clients automatically receive metadata updates for themselves (excluding changes they make themselves), channels they are joined to, and other clients in the channels they are joined to. If the `metadata-notify` capability is requested, clients also receive metadata updates for the users they are currently monitoring.

-----

If a channel/user they are receiving updates for changes one of the keys the client is subscribed to, they will receive a [`METADATA` message](#metadata-server-message) notifying them of the change. Clients MAY also receive metadata notifications for keys they have not subscribed to, or even when they have not subscribed to any keys.

Here are additional cases where clients will receive `METADATA` messages:

- Upon connecting to the server, clients receive their non-transient metadata (for example, metadata stored by the server or by services). If none exists, the server MUST send a `RPL_METADATAEND` message instead.
- When subscribing to a key, clients SHOULD receive the current value of that key for channels/users they are receiving updates for.
- If `metadata-notify` is negotiated, clients SHOULD receive the current values of keys they are subscribed to when they [`MONITOR`](https://ircv3.net/specs/extensions/monitor.html#monitor-command) a user.

## Postponed synchronization

If the client joins a large channel, or the client is already on some channels and enables the `metadata` capability, the server may not be able to send the client all current metadata for their targets.

In this case, the server MAY choose to respond with a `ERR_METADATASYNCLATER` numeric instead of propogating the current metadata of the targets. This numeric indicates that the specified target has some metadata set that the client SHOULD request synchronization of at a later time.

The client can use the [`SYNC`](#metadata-sync) subcommand to request the sync of metadata for the given target. If the `[<RetryAfter>]` is given, the client SHOULD wait at least that many seconds before sending the sync request.

## `metadata` Capability

The `metadata` capability indicates that a server supports metadata, and provides any limits and information about the system that clients must be aware of.

The ABNF format of the `metadata` capability is:

    capability ::= 'metadata' ['=' tokens]
    tokens     ::= token [',' token]*
    token      ::= key ['=' value]
    key        ::= <sequence of a-zA-Z0-9_.:- >
    value      ::= <utf8>

These are the defined tokens:

* `maxsub`: the maximum number of keys a client is allowed in its subscripion list. See the [`SUB`](#metadata-sub) subcommand for more details.
* `maxkey`: the maximum number of keys a client is allowed to set on its own nickname.

Clients MUST silently ignore any unknown tokens.

## Keys and Values

Key names MUST be restricted to the ranges `A-Z`, `a-z`, `0-9`, and `_.:-` and are case-insensitive. Key names MUST NOT start with a colon (`:`).

Values may take any form, but MUST be encoded using UTF-8.

The expected handling of individual metadata keys SHOULD be defined and listed in the IRCv3 extension registry.

## `METADATA` server message

Clients that request the `metadata` capability MUST be able to handle incoming `METADATA` messages. After negotiating this capability, servers MAY send this message to clients at any time.

The format of the `METADATA` server message is:

    METADATA <Target> <Key> <Visibility> <Value>

`Target` is a valid nickname or channel name.

`Key` is a valid key name

`Visibility` is an asterisk (`*`) for keys visible to everyone, or an implementation-defined value which describes the key's visibility status; for instance, it MAY be a permission level or flag. (TODO should we define a prefix like STATUSMSG for this value?)

`Value` is a UTF-8 encoded value.

## `METADATA` client command

    METADATA <Target> <Subcommand> [<Param 1> ... [<Param n>]]

`Target` is a valid nickname or channel name. Clients SHOULD use the asterisk symbol (`*`) when targeting their own nickname.

`Subcommand` is one of the following, described in detail (including any required params) further in the document:

* [GET](#todo)
* [LIST](#todo)
* [SET](#todo)
* [CLEAR](#todo)
* [SUB](#todo)
* [UNSUB](#todo)
* [SUBS](#todo)
* [SYNC](#todo)

## `METADATA` Subcommands

### METADATA GET

    METADATA <Target> GET key1 key2 ...

This subcommand lets clients lookup keys on the given target.

Multiple keys may be given.
The response will be either `RPL_KEYVALUE`, `ERR_KEYINVALID`
or `ERR_NOMATCHINGKEY` for every key in order.

Servers MAY replace metadata which is considered not visible for the requesting user, with `ERR_NOMATCHINGKEY` or with `ERR_KEYNOPERMISSION`.

*Errors*: `ERR_NOMATCHINGKEY`, `ERR_KEYINVALID`, `ERR_KEYNOPERMISSION`.

### METADATA LIST

    METADATA <Target> LIST

This subcommand lists all of the target's currently-set metadata keys along with their values.

If the target is valid, the response is zero or more `RPL_KEYVALUE` events, followed by a `RPL_METADATAEND` event.
If the target is not valid, ONLY the `ERR_TARGETINVALID` numeric is sent.

Servers MAY omit metadata which is considered not visible for the requesting user, or replace it with `ERR_KEYNOPERMISSION`.

*Errors*: `ERR_KEYNOPERMISSION`.

### METADATA SET

    METADATA <Target> SET <Key> [:Value]

This subcommand sets the key on the target to the given value. If no value is given, the key is removed.

If the key is invalid, the server responsds with `ERR_KEYINVALID` and fails the request.

If the key is valid, but not set, and the client tries to remove the key, the server responds with `ERR_KEYNOTSET` and fails the request.

If the user cannot set keys on the given target, the server responds with `ERR_KEYNOPERMISSION` and fails the request.

Servers MAY respond to certain keys considered not settable by the requesting user, or otherwise disallowed by the server, with `ERR_KEYNOPERMISSION` and fail the request.

Servers MAY respond with `ERR_METADATARATELIMIT` and fail the request. When a client receives `ERR_METADATARATELIMIT`, it SHOULD retry the `METADATA SET` request at a later time. If the `ERR_METADATARATELIMIT` event contains the OPTIONAL `<RetryAfter>` parameter, the parameter value MUST be a positive integer indicating the minimum number of seconds the client should wait before retrying the request.

If the request is successful, the server carries out the requested change and responds with one `RPL_KEYVALUE` event, representing the new value (or lack of one), and one `RPL_METADATAEND` event.

*Errors*: `ERR_METADATALIMIT`, `ERR_KEYINVALID`, `ERR_KEYNOTSET`, `ERR_KEYNOPERMISSION`, `ERR_METADATARATELIMIT`

### METADATA CLEAR

    METADATA <Target> CLEAR

This subcommand removes all metadata from the target, equivalently to using `METADATA SET` on all currently-set keys with an empty value.

If the user cannot clear keys on the given target, the server responds with `ERR_KEYNOPERMISSION` with an asterisk (`*`) in the `<Key>` field and fails the request.

Servers MAY omit certain keys which are considered not settable by the requesting user, or respond with `ERR_KEYNOPERMISSION` for each of those keys.

If the request is successful, the server responds with one `RPL_KEYVALUE` event per cleared key and then one `RPL_METADATAEND` event.

*Errors*: `ERR_KEYNOPERMISSION`

### METADATA SUB

    METADATA * SUB <key1> [<key2> ...]

Clients use this subcommand to subscribe to updates for the given keys.

The server MUST process each key in order, as the client uses this order to determine which keys they were and were not able to subscribe to. Clients SHOULD send keys in order of preference.

The server processes each key in order, and:

> If the client has subscribed to too many keys, the server does not process any further keys in the subcommand. The `<key>` parameter of the `ERR_METADATATOOMANYSUBS` numeric MUST be this first key that the client could not subscribe to (so the client knows all keys sent after that one were not processed).
>
> If the client successfully subscribes to a key, or is already subscribed to a requested key, that key MUST appear in a `RPL_METADATASUBOK` reply numeric.
>
> If the key's name is invalid, the server sends a `ERR_KEYINVALID` numeric to the client and continues processing keys.
>
> If the client does not have permission to view a given key, the server sends a `ERR_KEYNOPERMISSION` numeric to the client and continues processing keys. However, the subscription MUST still be successful, and that key MUST appear in a `RPL_METADATASUBOK` reply numeric. In this case, the `ERR_KEYNOPERMISSION` numeric serves as a warning indicating that the client will not receive `METADATA` messages about this key unless it gains the necessary (implementation defined) privileges later.

Once the server is finished processing keys, it responds with zero or more of these numerics in any order: `RPL_METADATASUBOK`, `ERR_KEYINVALID`, `ERR_KEYNOPERMISSION`, and MAY respond with one `ERR_METADATATOOMANYSUBS` numeric. Finally, the server ends the reply with one `RPL_METADATAEND` numeric.

### METADATA UNSUB

    METADATA * UNSUB <key1> [<key2> ...]

Clients use this subcommand to unsubscribe from updates for the given keys.

Servers process the given keys, and:

> If the client successfully unsubscribes from a key, or is not subscribed to a requested key, that key MUST appear in a `RPL_METADATAUNSUBOK` reply numeric.
>
> If the key's name is invalid, the server sends a `ERR_KEYINVALID` numeric to the client and continues processing keys.

Once the server is finished processing keys, it responds with zero or more of these numerics in any order: `RPL_METADATAUNSUBOK`, `ERR_KEYINVALID`. Finally, the server ends the reply with one `RPL_METADATAEND` numeric.

### METADATA SUBS

    METADATA * SUBS

This subcommand returns which keys the client is currently subscribed to.

The server responds with zero or more `RPL_METADATASUBS` numerics and then one `RPL_METADATAEND` numeric. The server MAY return the keys in any order. The server MUST NOT list the same key multiple times in a response to this subcommand.

### METADATA SYNC

    METADATA <Target> SYNC

Clients use this subcommand to receive all subscribed metadata from the given target.

If the sync cannot be performed at this time (due to load or other implementation-defined details), the server responds with a `ERR_METADATASYNCLATER` numeric. If the sync can be performed, the server responds with zero or more METADATA events.

For details, please see the [postponed synchronization](#postponed-synchronization) section.

## Numerics

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

Reference table of numerics and the `METADATA` subcommands or any other commands that produce them:

| Label                     | GET | LIST | SET | CLEAR | SUB | UNSUB | SUBS | SYNC | Other   |
| --- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
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

Each subcommand section describes the reply and error numerics it expects from the server, but here are brief descriptions of numerics that are used for multiple subcommands:

Replies:

* `RPL_KEYVALUE` reports the values of metadata keys. The `Visibility` parameter is defined in the [server message](#todo) section.
* `RPL_METADATAEND` delimits the end of a sequence of metadata replies.

Errors:

* `ERR_TARGETINVALID` when a client refers to an invalid target.
* `ERR_KEYINVALID` when a client refers to an invalid key.
* `ERR_KEYNOPERMISSION` when a client attempts to access or set a key on a target when they lack sufficient permission.

### `RPL_WHOISKEYVALUE` numeric

When a user runs `WHOIS` on a user with metadata, a subset of that metadata MAY be sent with `RPL_WHOISKEYVALUE` numerics. This subset MUST be chosen explicitly, and optimised for keys that can be easily read by users. For a complete view of user metadata, see the [`LIST`](#metadata-list) subcommand.

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

* In older, deprecated versions of the Metadata specification, the `metadata-notify` key subscribed you to all keys. Since we have now added the [`SUB`](#metadata-sub) and [`UNSUB`](#metadata-unsub) subcommands, the capability no longer acts in this way.
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