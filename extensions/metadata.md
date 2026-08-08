---
title: Metadata
layout: spec
redirect_from:
  - /specs/core/metadata-3.2.html
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
  -
    name: "Valerie Pond"
    period: "2022"
    email: "v.a.pond@outlook.com"
  -
    name: "Val Lorentz"
    period: "2022-2023"
    email: "progval+ircv3@progval.net"
  -
    name: "Ryan Schmidt"
    period: "2026"
    email: "moonmoon@libera.chat"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `metadata` capability name. Instead, implementations SHOULD use the `draft/metadata-3` capability name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

It is useful to associate metadata with one's IRC presence, e.g. to make one's homepage or non-IRC contact details more discoverable.
There are several mechanisms for doing this, but they normally rely on the presence of services and aren't really suitable for transient metadata like a user's current location.

This feature lays out a command that can be used to manage metadata associated with clients or channels and a notification framework to receive updates on metadata changes.

## Concepts

Clients and channels can both contain metadata. Metadata acts as a key:value store (for example, the `display-name` key on a user, or the `url` key on a channel).

Clients can set or delete metadata on themselves and on channels they administer. Privileged users (e.g. network admins) may be able to set certain metadata on other users, and set special keys on themselves or channels.

On joining a server, clients have to configure their 'metadata key subscriptions'. This is a list of which keys the client understands and wants to get updates about. For example, they may subscribe to a `status` key if they support user-set statuses. By default, this subscription list is empty. When metadata is updated or removed, users who have subscribed to the changed key will receive messages notifying them of the change.

## Definitions

*Visible channels* or *visible users* (generally, *visible target*) mean channels or users that a client would receive [metadata notifications](#notifications) about.

Servers may additionally restrict the visibility of metadata keys to certain subsets of users (e.g. based on operator status). A *visible key* is a metadata key that the user has permissions to view, according to the server's implementation-defined means.

A *metadata-aware client* is a client who has negotiated both the `draft/metadata-3` capability and the `batch` capability.

## Relation with other specifications

This specification depends on the [`batch`](../extensions/batch.html) capability which MUST be negotiated to use `draft/metadata-3`. The order of capability negotiation is not significant and MUST NOT be enforced.

This specification also uses the [standard replies](../extensions/standard-replies.html) framework.

Clients MUST NOT request more than one of `metadata-notify`, `draft/metadata-2`, and `draft/metadata-3`. Servers MUST NOT accept these requests either.

## `draft/metadata-3` Capability

The `draft/metadata-3` capability indicates that a server supports metadata, and provides any limits and information about the system that clients must be aware of. Clients MUST request this capability in order to receive [`METADATA` notifications](#notifications).

The ABNF format of the `draft/metadata-3` capability is:

    capability ::= 'draft/metadata-3' ['=' tokens]
    tokens     ::= token [',' token]*
    token      ::= key ['=' value]
    key        ::= <sequence of one or more a-z0-9_./->
    value      ::= <utf8>

These are the defined tokens:

* `before-connect`: If present, indicates the server supports `METADATA` commands during connection registration. This token has no value and clients MUST ignore any value sent with it.
* `max-subs`: The maximum number of keys a client is allowed in its subscription list. See the [`METADATA SUB`](#metadata-sub) command for more details.
* `max-keys`: The maximum number of keys a client is allowed to set on its own nickname.
* `max-key-bytes`: The maximum size of keys a client is allowed to set. Servers MAY send longer keys.
* `max-value-bytes`: The maximum size of values a client is allowed to set. Servers MAY send longer values.

Clients MUST silently ignore any unknown tokens. If the server does not provide a token indicating one of the above maximum limits, clients SHOULD assume that particular item has no limit.

If a client removes this capability or the batch capability from themselves, servers MUST clear that client's subscription list.

### Examples

*This section is non-normative.*

In both of these examples, max-subs is specified but other tokens are not. Clients should assume that max-keys, max-key-bytes, and max-value-bytes have no limit. In practice, the key and value must still fit on a single IRC protocol line despite not having any server-specified limit.

    C: CAP LS 302
    S: CAP * LS :userhost-in-names draft/metadata-3=foo,max-subs=50,bar multi-prefix
    C: CAP LS 302
    S: CAP * LS :draft/metadata-3=max-subs=25 multi-prefix invite-notify

## Keys and Values

Key names are restricted to the ranges `a-z`, `0-9`, and `_./-`. The empty string is an invalid value.

Values can take any form, but MUST be encoded using UTF-8. The empty string is an invalid value.

The expected handling of individual metadata keys SHOULD be [defined and listed in the IRCv3 extension registry](https://ircv3.net/registry).

## Batch types

This specification adds the `metadata` and `metadata-subs` batch types.

The `metadata` batch type takes one parameter and clients MUST ignore extra parameters. This parameter contains the target for which the metadata was requested, allowing the client to correctly process empty batches. Clients MUST NOT assume that all metadata in the batch applies to the entity referenced in the batch parameter; for example, a batch sent in response to a channel join or [`METADATA SYNC`](#metadata-sync) on a channel may contain metadata on users as well as the channel itself. Servers MAY send one or more of any of the following messages inside of this batch type:

* `761 RPL_KEYVALUE`
* `766 RPL_KEYNOTSET`
* `NOTE METADATA`
* `WARN METADATA`
* `FAIL METADATA`

The `metadata-subs` batch type takes no parameters and clients MUST ignore any parameters sent. Servers MAY send one or more of any of the following messages inside of this batch type:

* `772 RPL_METADATASUBS`
* `NOTE METADATA`
* `WARN METADATA`
* `FAIL METADATA`

## Replies

The following numerics are reserved for metadata, with these labels and parameters:

| No. | Label                     | Parameters                                       |
| --- | ------------------------- | ------------------------------------------------ |
| 760 | `RPL_WHOISKEYVALUE`       | `<Target> <Key> <Visibility> :<Value>`           |
| 761 | `RPL_KEYVALUE`            | `<Target> <Key> <Visibility> :<Value>`           |
| 766 | `RPL_KEYNOTSET`           | `<Target> <Key> :Key not set`                    |
| 770 | `RPL_METADATASUBOK`       | `<Key1> [<Key2> ...]`                            |
| 771 | `RPL_METADATAUNSUBOK`     | `<Key1> [<Key2> ...]`                            |
| 772 | `RPL_METADATASUBS`        | `<Key1> [<Key2> ...]`                            |
| 774 | `ERR_METADATASYNCLATER`   | `<Target> <RetryAfter> :Try again later`         |
| 775 | `ERR_METADATARATELIMIT`   | `<RetryAfter> <Target> <Command> [<Params> ...]` |

The trailing parameter of `766 RPL_KEYNOTSET` and `774 ERR_METADATASYNCLATER` MAY vary wildly between implementations.

The `<RetryAfter>` parameter of `774 ERR_METADATASYNCLATER` and `775 ERR_METADATARATELIMIT` MUST be a non-negative integer. This parameter indicates the number of seconds clients SHOULD wait before re-trying the command.

The `<Visibility>` parameter of `760 RPL_WHOISKEYVALUE` and `761 RPL_KEYVALUE` is an asterisk (`*`) for keys visible to everyone, or an implementation-defined value which describes the key’s visibility status; for instance, it MAY be a permission level or flag.

The `<Target>`, `<Command>`, and `<Params>` parameters of `775 ERR_METADATARATELIMIT` reflect the target, subcommand, and any additional parameters that were passed into the rate-limited command, so that clients can re-send the command without needing stateful tracking of what was originally sent.

The following [standard reply](../extensions/standard-replies.html) codes are used in this specification, with these labels and parameters:

| Code                    | Parameters                                     |
| ----------------------- | ---------------------------------------------- |
| `INVALID_KEY`           | `<Key> :Invalid key`                           |
| `INVALID_PARAMS`        | `<Command> :Invalid parameters`                |
| `INVALID_TARGET`        | `<Target> :Invalid metadata target`            |
| `INVALID_VALUE`         | `<Key> :Invalid value`                         |
| `KEY_NO_PERMISSION`     | `<Target> <Key> :Permission denied`            |
| `LIMIT_REACHED`         | `<Command> <Item> <Limit> :Limit reached`      |

The trailing parameter containing the descriptive message MAY vary wildly between implementations and SHOULD contain a relevant description for the specific error encountered.

The `<Command>` parameter of `INVALID_PARAMS` and `LIMIT_REACHED` is the subcommand specified by the client, or an asterisk (`*`) if no subcommand was provided or the subcommand is not valid as a middle parameter.

The `<Item>` parameter of `LIMIT_REACHED` will be the target if the metadata limit for a particular target was reached while performing [`METADATA SET`](#metadata-set) or the key name unable to be subscribed to if the subscription limit was reached while performing [`METADATA SUB`](#metadata-sub).

## Notifications

By default, clients are not subscribed to any keys. They can subscribe and unsubscribe from keys using the [`SUB`](#metadata-sub) and [`UNSUB`](#metadata-unsub) subcommands, respectively. Servers MUST allow clients to subscribe to any valid keys, including privileged keys the client cannot access at the time of subscription.

When a client is subscribed to a key, they automatically receive metadata updates regarding that key. These updates will be either a `761 RPL_KEYVALUE` indicating the key on the target was set to a particular value or a `766 RPL_KEYNOTSET` indicating that the key was removed from the target. Clients MAY also receive metadata notifications for keys they have not subscribed to, or even when they have not subscribed to any keys.

Servers MUST send notifications to subscribers the key is visible to for changes made:

* to that subscriber (excluding changes the subscriber made on themselves),
* to channels that subscriber is joined to,
* to other clients in the channels that subscriber is joined to, and
* to users that subscriber is currently monitoring.

## Automatic synchronization

The notification framework provides clients updates on metadata keys for visible targets, but does not share any information about keys already set before the target became visible to the client. This section describes events outside of the `METADATA` command that trigger automatic synchronization of metadata so that the client has a usable baseline to work from.

If the `draft/metadata-3` capability was negotiated during connection registration, servers MUST send clients a list of their current metadata (any metadata stored by the server or by services, plus any metadata set by the client during connection registration via `before-connect`) in a `metadata` batch with their own nick as target as part of the registration burst, i.e. before `RPL_ENDOFMOTD` or `ERR_NOMOTD`. This batch MUST include all visible keys set on the client, irrespective of whether the client has subscribed to that key.

When server sends `730 RPL_MONONLINE` to a metadata-aware client (see [`MONITOR`](../extensions/monitor.html#monitor-command)), the server MUST additionally send either a `metadata` batch containing the monitored user's metadata to the client or a `774 ERR_METADATASYNCLATER` message to indicate postponed synchronization. If the server sends a batch, the batch MUST be identical to what would be returned if [`METADATA SYNC`](#metadata-sync) was issued against that target.

When a metadata-aware client joins a channel, the server MUST send either a `metadata` batch containing the channel's metadata as well as the metadata for all users on the channel or a `774 ERR_METADATASYNCLATER` message to indicate postponed synchronization. If the server sends a batch, the batch MUST be identical to what would be returned if [`METADATA SYNC`](#metadata-sync) was issued against that channel.

## Postponed synchronization

Servers MAY respond with `774 ERR_METADATASYNCLATER` to certain types of automatic synchronization as well as to the [`METADATA SYNC`](#metadata-sync) command for implementation-defined reasons, such as joining a channel with too many users or internal rate-limiting. This numeric indicates that the client SHOULD request synchronization of that target's metadata at a later time.

The client can use the `METADATA SYNC` command to request the synchronization of metadata for the given target. If the `<RetryAfter>` parameter of the numeric is nonzero, the client SHOULD wait at least that many seconds before sending the synchronization request.

### Examples

*This section is non-normative.*

These examples assume the client has previously enabled the [no-implicit-names](../extensions/no-implicit-names.html) capability and has subscribed to the `display-name` metadata key for brevity.

A metadata-aware client joins multiple channels and the server responds with `774 ERR_METADATASYNCLATER` to each of those joins with a target of `*ALL`. The client deduplicates these and sends only one `METADATA *ALL SYNC` after all channel joins are complete. This server implementation additionally responds with `766 RPL_KEYNOTSET` for subscribed keys that are not set on the target or are not visible to the user.

    C: JOIN #chan1,#chan2,#chan3,#chan4
    S: :modernclient!modernclient@example.com JOIN #chan1
    S: :irc.example.com 774 modernclient * 0 :Please synchronize metadata later
    S: :modernclient!modernclient@example.com JOIN #chan2
    S: :irc.example.com 774 modernclient * 0 :Please synchronize metadata later
    S: :modernclient!modernclient@example.com JOIN #chan3
    S: :irc.example.com 774 modernclient * 0 :Please synchronize metadata later
    S: :modernclient!modernclient@example.com JOIN #chan4
    S: :irc.example.com 774 modernclient * 0 :Please synchronize metadata later
    C: METADATA *ALL SYNC
    S: BATCH +r2Nla metadata *ALL
    S: @batch=r2Nla :irc.example.com 761 modernclient #chan1 display-name * :channel 1️⃣
    S: @batch=r2Nla :irc.example.com 761 modernclient #chan2 display-name * :channel 2️⃣
    S: @batch=r2Nla :irc.example.com 766 modernclient #chan3 display-name :Key not set
    S: @batch=r2Nla :irc.example.com 761 modernclient #chan4 display-name * :channel 4️⃣
    S: @batch=r2Nla :irc.example.com 761 modernclient userA display-name * :A User
    S: @batch=r2Nla :irc.example.com 766 modernclient userB display-name :Key not set
    S: @batch=r2Nla :irc.example.com 761 modernclient userC display-name * :C User
    ... and many more messages
    S: BATCH -r2Nla

A metdata-aware client joins a large channel and the server responds with `774 ERR_METADATASYNCLATER` with a defined delay. After the initial delay, the server is still not ready to process the command. This server implementation additionally omits subscribed keys that are not set on the target or are not visible to the user.

    C: JOIN #bigchannel
    S: :modernclient!modernclient@example.com JOIN #bigchannel
    S: :irc.example.com 774 modernclient #bigchannel 4 :Please synchronize metadata later
    -- 4 seconds later --
    C: METADATA #bigchannel SYNC
    S: :irc.example.com 774 modernclient #bigchannel 6 :Please synchronize metadata later
    -- 6 more seconds later --
    C: METADATA #bigchannel SYNC
    S: BATCH +M40ad metadata #bigchannel
    S: @batch=M40ad :irc.example.com 761 modernclient userA display-name * :A User
    S: @batch=M40ad :irc.example.com 761 modernclient userC display-name * :C User
    ... and many more messages
    S: BATCH -M40ad

## METADATA client command

    METADATA <Target> <Subcommand> [<Param 1> ... [<Param n>]]

`Target` is a valid nickname or channel name. Clients SHOULD use the asterisk symbol (`*`) when targeting their own nickname.

`Subcommand` is one of the subcommands listed below. The allowed params are described in each subcommand description.

Clients MAY use this command during connection registration if the server advertises the `before-connect` token. Clients that have not completed connection registration MUST use `*` to target themselves, since they have not been assigned a nickname, even if they sent a `NICK` command.

If the server receives a syntactically invalid `METADATA` command, e.g., an unknown subcommand, missing parameters, or excess parameters, the server SHOULD reply with `FAIL METADATA INVALID_PARAMS`. If an existing error numeric such as `461 ERR_NEEDMOREPARAMS` is appropriate, it MAY be used instead.

Servers MAY rate limit any `METADATA` subcommand. If they do so and a client is rate limited, servers SHOULD send `775 ERR_METADATARATELIMIT` indicating the client SHOULD retry the `METADATA` request at a later time.

### Examples

*This section is non-normative.*

    C: METADATA
    S: FAIL METADATA INVALID_PARAMS * :Not enough parameters
    C: METADATA #chan GET
    S: FAIL METADATA INVALID_PARAMS GET :Not enough parameters
    C: METADATA #chan SNEEZE
    S: FAIL METADATA INVALID_PARAMS SNEEZE :Unknown subcommand "SNEEZE"
    C: METADATA #chan :: hi!
    S: FAIL METADATA INVALID_PARAMS * :Unknown subcommand ": hi!"

### METADATA GET

    METADATA <Target> GET <Key 1> [<Key 2> ... [<Key n>]]

This subcommand lets clients look up the specified keys on the given target. If the target is valid, the response will be a `metadata` batch containing one of the following responses for each key specified by the client:

* `761 RPL_KEYVALUE` with the key's value
* `766 RPL_KEYNOTSET` indicating the key is not set or the key is not visible to the requesting client
* `WARN METADATA INVALID_KEY` indicating the key contains invalid characters or is otherwise considered invalid by the server
* `WARN METADATA KEY_NO_PERMISSION` indicating the key is not visible to the requesting client

The ordering of responses within the batch MUST match the order keys were specified by the client. In the event that a key is not visible to the requesting client, the server MAY choose to send either `766 RPL_KEYNOTSET` or `WARN METADATA KEY_NO_PERMISSION`, but MUST NOT send both.

If the target is not valid, the server MUST reply with `FAIL METADATA INVALID_TARGET` instead of a batch.

### METADATA LIST

    METADATA <Target> LIST

This subcommand lists all of the target's currently-set metadata keys along with their values. If the target is valid, the response is a `metadata` batch containing `761 RPL_KEYVALUE` for all visible keys. Servers MAY additionally include information about keys that are not visible to the client by sending `WARN METADATA KEY_NO_PERMISSION` for such keys.

If the target is not valid, the server MUST reply with `FAIL METADATA INVALID_TARGET` instead of a batch.

### METADATA SET

    METADATA <Target> SET <Key> [:<Value>]

This subcommand sets the key on the target to the given value. If the Value parameter is omitted or is an empty string, the key is removed. If the request is successful, the server carries out the requested change and responds with one `761 RPL_KEYVALUE` event, representing the new value if any, or `766 RPL_KEYNOTSET` if no value is set. This new value MAY differ from the one sent by the client.

Otherwise, the server MUST send a `FAIL METADATA` response with an appropriate error message:

* `FAIL METADATA INVALID_TARGET` if the specified target is not valid
* `FAIL METADATA INVALID_KEY` if the key contains invalid characters or is otherwise deemed invalid by the server
* `FAIL METADATA KEY_NO_PERMISSION` if the client does not have permission to set keys on the specified target or is otherwise disallowed from setting or removing this key by the server
* `FAIL METADATA LIMIT_REACHED` if trying to add a new key when the maximum number of keys has already been set on the target

It is not an error if a client attempts to set a key to its current value or remove a nonexistent key; such attempts generate successful responses if there are no other issues with the request.

### METADATA CLEAR

    METADATA <Target> CLEAR

This subcommand removes all metadata from the target, equivalently to using `METADATA SET` on all currently-set keys with no value. If the user cannot clear keys on the given target, the server responds with `FAIL METADATA KEY_NO_PERMISSION` with an asterisk (`*`) in the `<Key>` field and fails the request.

If the request is successful, the server responds with a `metadata` batch containing one `766 RPL_KEYNOTSET` message per cleared key. Servers MAY omit certain keys which are considered not settable by the requesting user or respond with `WARN METADATA KEY_NO_PERMISSION` for each of those keys.

### METADATA SUB

    METADATA <Target> SUB <Key 1> [<Key 2> ... [<Key n>]]

This subcommand lets clients subscribe to updates for the given keys. If the `<Target>` parameter is not an asterisk (`*`) or the client's nickname, the server MUST fail the request with `FAIL METADATA INVALID_TARGET`. Otherwise, the server MUST process each key in order and clients SHOULD send keys in order of preference.

The server responds with one or more of the following responses:

* `770 RPL_METADATASUBOK` for successful subscriptions or when the client is already subscribed to that key,
* `WARN METADATA INVALID_KEY` for keys that contain invalid characters or are otherwise deemed as invalid by the server, or
* `FAIL METADATA LIMIT_REACHED` if the client has reached the maximum allowable number of subscriptions. If the server sends this, it MUST stop processing any further keys. The `<Item>` parameter of the `FAIL METADATA LIMIT_REACHED` reply MUST be this first key that the client could not subscribe to (so the client knows all keys sent after that one were not processed).

If the client does not have permission to view a given key, the server MAY send a `NOTE METADATA KEY_NO_PERMISSION` reply to the client. The subscription MUST still be successful and that key MUST appear in a `770 RPL_METADATASUBOK` reply. In this case, the `NOTE METADATA KEY_NO_PERMISSION` reply serves as a notice indicating that the client will not receive notifications about this key unless it gains the necessary (implementation-defined) privileges later.

After finishing sending responses as described above, if the client has completed connection registration and has added new subscriptions, the server MUST then either send a `metadata` batch with `*ALL` as its parameter containing the current values of all newly-subscribed keys visible to the user for all visible targets or a `774 ERR_METADATASYNCLATER` message with `*ALL` as its `<Target>` to indicate [postponed synchronization](#postponed-synchronization). In the event that the client has also negotiated [`labeled-response`](../extensions/labeled-response.html), the `metadata` batch MUST be nested within the `labeled-response` batch.

### Examples

*This section is non-normative.*

The client is joined to #chan (sharing the channel with userA, userB, and userC) and additionally has userD on their `MONITOR` list, who is currently online. The server is configured to allow a maximum of 3 subscriptions (advertising `max-subs=3` in the `draft/metadata-3` CAP value).

    C: METADATA * SUB display-name status super-secret-key avatar
    S: :irc.example.com 770 modernclient display-name status super-secret-key
    S: :irc.example.com NOTE METADATA KEY_NO_PERMISSION SUB * super-secret-key :You do not have permission to view this key
    S: :irc.example.com FAIL METADATA LIMIT_REACHED SUB avatar 3 :Cannot subscribe to "avatar" as you have reached the subscription limit
    S: :irc.example.com BATCH +d8amD metadata *
    S: @batch=d8amD :irc.example.com 761 modernclient #chan display-name :Cool Channel
    S: @batch=d8amD :irc.example.com 761 modernclient userA display-name :User A
    S: @batch=d8amD :irc.example.com 761 modernclient userA status :Playing games
    S: @batch=d8amD :irc.example.com 761 modernclient userD display-name :User D
    S: :irc.example.com BATCH -d8amD

### METADATA UNSUB

    METADATA <Target> UNSUB <Key 1> [<Key 2> ... [<Key n>]]

This subcommand lets clients remove their subscriptions of the given keys. If the `<Target>` parameter is not an asterisk (`*`) or the client's nickname, the server MUST fail the request with `FAIL METADATA INVALID_TARGET`. Otherwise, the server MUST process each key in order and clients SHOULD send keys in order of preference.

Servers process the given keys, and:

The server responds with one of the following responses for each key:

* `771 RPL_METADATAUNSUBOK` for successful removal of subscriptions or when the client is not subscribed to that key, or
* `WARN METADATA INVALID_KEY` for keys that contain invalid characters or are otherwise deemed as invalid by the server.

### METADATA SUBS

    METADATA <Target> SUBS

This subcommand returns which keys the client is currently subscribed to. If the `<Target>` parameter is not an asterisk (`*`) or the client's nickname, the server MUST fail the request with `FAIL METADATA INVALID_TARGET`.

The server responds with a `metadata-subs` batch containing zero or more `772 RPL_METADATASUBS` messages. The server MAY return the keys in any order. The server MUST NOT list the same key multiple times in a response to this subcommand.

If the client does not have permission to view a given subscribed key, the server MAY additionally send a `NOTE METADATA KEY_NO_PERMISSION` reply to the client within the `metadata-subs` batch for that key. This reply serves as a notice indicating that the client will not receive notifications about this key unless it gains the necessary (implementation-defined) privileges later.

### METADATA SYNC

    METADATA <Target> SYNC

Clients use this subcommand to receive all subscribed metadata from the given target. If the target is a channel, it also synchronizes the metadata for all other users in that channel. If the target is an asterisk followed by uppercase "ALL" (`*ALL`), it synchronizes metadata for all visible targets.

If the sync cannot be performed at this time (due to load or other implementation-defined details), the server responds with a `774 ERR_METADATASYNCLATER` indicating [postponed synchronization](#postponed-synchronization). If the sync can be performed, the server responds with a `metadata` batch containing zero or more of the following messages for each key the client is subscribed to:

* `761 RPL_KEYVALUE` with the key's value
* `766 RPL_KEYNOTSET` indicating the key is not set or the key is not visible to the requesting client
* `WARN METADATA KEY_NO_PERMISSION` indicating the key is not visible to the requesting client

In the event that a key is not visible to the requesting client, the server MAY choose to send either `766 RPL_KEYNOTSET` or `WARN METADATA KEY_NO_PERMISSION`, or it MAY omit the key entirely from the batch, but it MUST NOT send both.

The batch MAY additionally contain messages for keys that the client is not subscribed to.

## Client implementation considerations

*This section is non-normative.*

While this is true of any batch, clients should take particular care not to pause processing of other messages while a `metadata` batch is open. As these batches can be potentially large, servers are likely to produce them asynchronously in order to avoid freezing delivery of more important messages or filling up a client's send queue.

As servers may rewrite values set by clients with `METADATA SET`, clients should check the response before storing it in any local cache.

Because servers may choose to omit unset keys or keys that are not visible to the client from `metadata` batches, clients should assume that any subscribed keys missing from a `METADATA SYNC` have been unset in the interim and update local caches accordingly.

Avoid sending more than 12 keys with `METADATA GET`, `METADATA SUB`, or `METADATA UNSUB` unless you know the server is capable of handling more parameters. Old RFCs limited the number of parameters in an IRC message to 15, and some server implementations treat the command itself towards that parameter limit.

Many clients can be configured to automatically join channels upon connect. If a client receives `774 ERR_METADATASYNCLATER` while performing such automatic joins, it should consider ignoring such numerics and instead send a single `METADATA *ALL SYNC` after all automatic joins have been completed. Doing this allows for server-side deduplication of user metadata for users that are present in more than one of the automatically-joined channels and may yield faster results depending on server rate-limiting implementations.

## Server implementation considerations

*This section is non-normative.*

Servers with traditional send queue setups should ensure that they do not disconnect clients requesting large `metadata` batches, such as by producing the batch asynchronously. In cases where such a process is not feasible (such as most automatic synchronization batches), they should make use of postponed synchronization when the amount of metadata to send is too large (by whatever server-specific metric is chosen).

Servers may wish to provide network administrators the ability to configure an allowlist or denylist of metadata keys. They may also wish to reserve certain keys or patterns of keys as requiring elevated permissions to view or modify, to allow network administrators the ability to use `METADATA` as a secure key:value store for network operations or to ensure certain keys are only writable by services.

When responding with `774 ERR_METADATASYNCLATER` or `775 ERR_METADATARATELIMIT`, servers should attempt to provide a usable value for the `<RetryAfter>` parameter, to avoid clients flooding the server with retries. A server is not required to accept the command after the provided RetryAfter period and may still rate-limit the client at that time. This provides a middle ground for servers that do not wish to publish precise rate-limit details to clients while ensuring clients do not attempt to retry commands too aggressively.

## Reference tables

Reference table of numerics and the `METADATA` subcommands or any other commands that produce them:

| Label                              | GET | LIST | SET | CLEAR | SUB | UNSUB | SUBS | SYNC | Other             |
| ---------------------------------- | :-: | :--: | :-: | :---: | :-: | :---: | :--: | :--: | :---------------: |
| `RPL_WHOISKEYVALUE`                |     |      |     |       |     |       |      |      | `WHOIS`           |
| `RPL_KEYVALUE`                     | *   | *    | *   |       | *   |       |      | *    |                   |
| `RPL_KEYNOTSET`                    | *   |      | *   | *     | *   |       |      | *    |                   |
| `RPL_METADATASUBOK`                |     |      |     |       | *   |       |      |      |                   |
| `RPL_METADATAUNSUBOK`              |     |      |     |       |     | *     |      |      |                   |
| `RPL_METADATASUBS`                 |     |      |     |       |     |       | *    |      |                   |
| `ERR_METADATASYNCLATER`            |     |      |     |       | *   |       |      | *    | `JOIN`, `MONITOR` |
| `ERR_METADATARATELIMIT`            | *   | *    | *   | *     | *   | *     | *    | *    |                   |

Reference table of Standard Replies codes and the `METADATA` subcommands or any other commands that produce them:

| Code                    | GET  | LIST | SET       | CLEAR | SUB  | UNSUB | SUBS | SYNC | Other   |
| ----------------------- | :--: | :--: | :-------: | :---: | :--: | :---: | :--: | :--: | :-----: |
| `INVALID_KEY`           | WARN |      | FAIL      |       | WARN | WARN  |      |      |         |
| `INVALID_PARAMS`        | FAIL | FAIL | FAIL      | FAIL  | FAIL | FAIL  | FAIL | FAIL | FAIL    |
| `INVALID_TARGET`        | FAIL | FAIL | FAIL      | FAIL  | FAIL | FAIL  | FAIL | FAIL |         |
| `INVALID_VALUE`         |      |      | FAIL      |       |      |       |      |      |         |
| `KEY_NO_PERMISSION`     | WARN | WARN | FAIL/WARN | WARN  | NOTE |       | NOTE | WARN |         |
| `LIMIT_REACHED`         |      |      | FAIL      |       | FAIL |       |      |      |         |

## Examples

*This section and all of its sub-sections are non-normative.*

All examples begin with the client not being subscribed to any keys.

### METADATA command examples

#### Setting metadata on self

    C: METADATA * SET url :http://www.example.com
    S: :irc.example.com 761 client * url * :http://www.example.com

#### Setting metadata on self, but the limit has been reached

    C: METADATA * SET url :http://www.example.com
    S: FAIL METADATA LIMIT_REACHED SET * 50 :Metadata limit reached

#### Setting metadata on another user, without permission

    C: METADATA user1 SET url :http://www.example.com
    S: FAIL METADATA KEY_NO_PERMISSION user1 url :You do not have permission to set 'url' on 'user1'

#### Setting metadata on channel

    C: METADATA #example SET url :http://www.example.com
    S: :irc.example.com 761 client #example url * :http://www.example.com

#### Setting metadata on an invalid target

    C: METADATA $a:user SET url :http://www.example.com
    S: FAIL METADATA INVALID_TARGET $a:user :Invalid target.

#### Setting metadata with an invalid key

    C: METADATA user1 SET $url$ :http://www.example.com
    S: FAIL METADATA INVALID_KEY $url$ :Invalid key.

#### Server rate-limits setting metadata

    C: METADATA * SET url :http://www.example.com
    S: :irc.example.com 775 client 5 * SET url :http://www.example.com

Servers can pick an arbitrary value here, if a client waits that long there is no guarantee the next attempt will not also be rate-limited.

-----

### Notification examples

#### Metadata was set on a user

    S: :irc.example.com 761 client user1 account * :user1

### Metadata was set on a channel

    S: :irc.example.com 761 client #example wiki-url * :http://wiki.example.com

-----

### Listing and Viewing Metadata Examples

#### Listing metadata, with an implementation-defined visibility field

    C: METADATA user1 LIST
    S: :irc.example.com BATCH +VUN2ot metadata
    S: @batch=VUN2ot :irc.example.com 761 client user1 url * :http://www.example.com
    S: @batch=VUN2ot :irc.example.com 761 client user1 im.xmpp * :user1@xmpp.example.com
    S: @batch=VUN2ot :irc.example.com 761 client user1 bot-likeliness-score visible-only-for-admin :42
    S: :irc.example.com BATCH -VUN2ot

#### Getting several metadata keys from a user

    C: METADATA user1 GET blargh splot im.xmpp
    S: :irc.example.com BATCH +gWkCiV metadata
    S: @batch=gWkCiV :irc.example.com 766 client user1 blargh :No matching key
    S: @batch=gWkCiV :irc.example.com 766 client user1 splot :No matching key
    S: @batch=gWkCiV :irc.example.com 761 client user1 im.xmpp * :user1@xmpp.example.com
    S: :irc.example.com BATCH -gWkCiV

#### Client joins a channel and syncs metadata immediately

    C: JOIN #smallchan
    S: :modernclient!modernclient@example.com JOIN #smallchan
    S: :irc.example.com 353 modernclient @ #smallchan :user1 user2 user3 user4 user5 ...
    S: :irc.example.com 353 modernclient @ #smallchan :user51 user52 user53 user54 ...
    S: :irc.example.com 353 modernclient @ #smallchan :user101 user102 user103 user104 ...
    S: :irc.example.com 353 modernclient @ #smallchan :user151 user152 user153 user154 ...
    S: :irc.example.com 366 modernclient #smallchan :End of /NAMES list.
    S: :irc.example.com BATCH +UwZ67M metadata
    S: @batch=UwZ67M :irc.example.com 761 modernclient user2 bar * :second example value 
    S: @batch=UwZ67M :irc.example.com 761 modernclient user1 foo * :third example value
    S: @batch=UwZ67M :irc.example.com 761 modernclient user1 bar * :this is another example value
    S: @batch=UwZ67M :irc.example.com 761 modernclient user3 website * :www.example.com
    ...and some more metadata messages
    S: :irc.example.com BATCH -UwZ67M

#### Client joins a channel and but needs to sync metadata later

Client joins channel:

    C: JOIN #bigchan
    S: :modernclient!modernclient@example.com JOIN #bigchan
    S: :irc.example.com 353 modernclient @ #bigchan :user1 user2 user3 user4 user5 ...
    S: :irc.example.com 353 modernclient @ #bigchan :user51 user52 user53 user54 ...
    S: :irc.example.com 353 modernclient @ #bigchan :user101 user102 user103 user104 ...
    S: :irc.example.com 353 modernclient @ #bigchan :user151 user152 user153 user154 ...
    S: :irc.example.com 366 modernclient #bigchan :End of /NAMES list.
    S: :irc.example.com 774 modernclient #bigchan 4 :Please try SYNC again in 4 seconds.

Client waits 4 seconds:

    C: METADATA #bigchan SYNC
    S: :irc.example.com 774 modernclient #bigchan 6 :Please try SYNC again in 6 seconds.

Client waits 6 more seconds:

    C: METADATA #bigchan SYNC
    S: :irc.example.com BATCH +O5J6rk metadata
    S: @batch=O5J6rk :irc.example.com 761 modernclient user52 foo * :example value 1
    S: @batch=O5J6rk :irc.example.com 761 modernclient user2 bar * :second example value 
    S: @batch=O5J6rk :irc.example.com 761 modernclient user1 foo * :third example value
    S: @batch=O5J6rk :irc.example.com 761 modernclient user1 bar * :this is another example value
    S: @batch=O5J6rk :irc.example.com 761 modernclient user152 baz * :Lorem ipsum
    S: @batch=O5J6rk :irc.example.com 761 modernclient user3 website * :www.example.com
    S: @batch=O5J6rk :irc.example.com 761 modernclient user152 bar * :dolor sit amet
    ...and many more metadata messages
    S: :irc.example.com BATCH -O5J6rk

-----

### Subscription Examples

Most replies to `METADATA SUB` in these examples send `774 ERR_METADATASYNCLATER` for brevity. A real implementation may wish to more eagerly send `metadata` batches in response to `METADATA SUB`.

#### Basic subscriping and unsubscribing

    C: METADATA * SUB avatar website foo bar
    S: :irc.example.com 770 modernclient avatar website foo bar
    S: :irc.example.com BATCH +vla9D metadata *ALL
    ...metadata messages for avatar/website/foo/bar for all visible targets
    S: :irc.exmaple.com BATCH -vla9D
    C: METADATA * UNSUB foo bar
    S: :irc.example.com 771 modernclient bar foo

#### Multiple `RPL_METADATASUBOK` numerics in reply to `METADATA SUB`

    C: METADATA * SUB avatar website foo bar baz
    S: :irc.example.com 770 modernclient avatar website
    S: :irc.example.com 770 modernclient foo
    S: :irc.example.com 770 modernclient bar baz
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later

#### Invalid key name in reply to subscription

    C: METADATA * SUB foo $url bar
    S: :irc.example.com 770 modernclient foo bar
    S: WARN METADATA INVALID_KEY $url :Invalid key
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later

#### "Subscribed to too many keys" error in reply to subscription 1

The client first successfully subscribes to some keys and later it tries to subscribe to some more keys, unsuccessfully.

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com 770 modernclient website avatar foo bar baz
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUB email city
    S: FAIL METADATA LIMIT_REACHED SUB email 5 :Too many subscriptions!
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +c2LO6 metadata-subs
    S: @batch=c2LO6 :irc.example.com 772 modernclient website avatar foo bar baz
    S: :irc.example.com BATCH -c2LO6

#### "Subscribed to too many keys" error in reply to subscription 2

This is like the previous case, except when the second METADATA SUB happens the server accepts the first 2 keys (`email`, `city`) but not the rest (`country`, `bar`, `baz`).

    C: METADATA * SUB website avatar foo
    S: :irc.example.com 770 modernclient website avatar foo
    C: METADATA * SUB email city country bar baz
    S: FAIL METADATA LIMIT_REACHED SUB country 5 :Too many subscriptions!
    S: :irc.example.com 770 modernclient email city
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +mda82 metadata-subs
    S: @batch=mda82 :irc.example.com 772 modernclient website avatar city foo email
    S: :irc.example.com BATCH -mda82

#### "Subscribed to too many keys" error in reply to subscription 3

In this case, the client is trying to subscribe to a key that it is already subscribed to (`website`), but the key is not processed because the limit imposed by the server on the number of subscribed keys is reached before the `website` key is processed by the server.
The client, however, successfully subscribes to the `foo` key which was also in the second request, but it appeared before the `website` key.

    C: METADATA * SUB avatar website
    S: :irc.example.com 770 modernclient avatar website
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUB foo website avatar
    S: FAIL METADATA LIMIT_REACHED SUB website 3 :Too many subscriptions!
    S: :irc.example.com 770 modernclient :foo
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +DMa04 metadata-subs
    S: @batch=DMa04 :irc.example.com 772 modernclient avatar foo website
    S: :irc.example.com BATCH -DMa04

#### Querying the list of subscribed keys 1

The server replies with a single `772 RPL_METADATASUBS` numeric.

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com 770 modernclient website avatar foo bar baz
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +TACM1 metadata-subs
    S: @batch=TACM1 :irc.example.com 772 modernclient avatar bar baz foo website
    S: :irc.example.com BATCH -TACM1

#### Querying the list of subscribed keys 2

The server replies with multiple `772 RPL_METADATASUBS` numerics.

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com 770 modernclient website avatar foo bar baz
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +B089a metadata-subs
    S: @batch=B089a :irc.example.com 772 modernclient avatar
    S: @batch=B089a :irc.example.com 772 modernclient bar baz
    S: @batch=B089a :irc.example.com 772 modernclient foo website
    S: :irc.example.com BATCH -B089a

#### Empty list of subscribed keys

In this case, there are no `772 RPL_METADATASUBS` numerics sent.

    C: METADATA * SUBS
    S: :irc.example.com BATCH +fad02 metadata-subs
    S: :irc.example.com BATCH -fad02

#### Unsubscribing

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com 770 modernclient website avatar foo bar baz
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +5r8af metadata-subs
    S: @batch=5r8af :irc.example.com 772 modernclient avatar bar baz foo website
    S: :irc.example.com BATCH -5r8af
    C: METADATA * UNSUB bar foo baz
    S: :irc.example.com 771 modernclient baz foo bar
    C: METADATA * SUBS
    S: :irc.example.com BATCH +b05mn metadata-subs
    S: @batch=b05mn :irc.example.com 772 modernclient avatar website
    S: :irc.example.com BATCH -b05mn

#### Subscribing to the same key multiple times 1

No `metadata` batch or `774 ERR_METADATASYNCLATER` response is given to the second `METADATA SUB` because no new keys were subscribed to.

    C: METADATA * SUB website avatar foo bar baz
    S: :irc.example.com 779 modernclient website avatar foo bar baz
    S: :irc.example.com BATCH +bv48i metadata *ALL
    S: ...metadata messages for website/avatar/foo/bar/baz for all visible targets
    S: :irc.example.com BATCH -bv48i
    C: METADATA * SUBS
    S: :irc.example.com BATCH +mz7qq metadata-subs
    S: @batch=mz7qq :irc.example.com 772 modernclient avatar bar baz foo website
    S: :irc.example.com BATCH -mz7qq
    C: METADATA * SUB avatar website
    S: :irc.example.com 770 modernclient avatar website
    C: METADATA * SUBS
    S: :irc.example.com BATCH +1mM45 metadata-subs
    S: @batch=1mM45 :irc.example.com 772 modernclient avatar bar baz foo website
    S: :irc.example.com BATCH -1mM45

#### Subscribing to the same key multiple times 2

The client (erroneously) subscribes to the same key twice in the same command.
The server is free to include the key being subscribed to in the `770 RPL_METADATASUBOK` numeric once or twice.

In both cases, the key will only appear once in the reply to a following `METADATA SUBS` command.

Once:

    C: METADATA * SUB avatar avatar
    S: :irc.example.com 770 modernclient avatar
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +MC02d metadata-subs
    S: @batch=MC02d :irc.example.com 772 modernclient avatar
    S: :irc.example.com BATCH -MC02d

Twice:

    C: METADATA * SUB avatar avatar
    S: :irc.example.com 770 modernclient avatar avatar
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +MC02d metadata-subs
    S: @batch=MC02d :irc.example.com 772 modernclient avatar
    S: :irc.example.com BATCH -MC02d

#### Unsubscribing from a non-subscribed key 1

    C: METADATA * SUBS
    S: :irc.example.com BATCH +mjay4 metadata-subs
    S: :irc.example.com BATCH -mjay4
    C: METADATA * UNSUB website
    S: :irc.example.com 771 modernclient website
    C: METADATA * SUBS
    S: :irc.example.com BATCH +a2OWA metadata-subs
    S: :irc.example.com BATCH -a2OWA
    C: METADATA * SUB website
    S: :irc.example.com 771 modernclient website
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +rbqDZ metadata-subs
    S: @batch=rbqDZ :irc.example.com 772 modernclient website
    S: :irc.example.com BATCH -rbqDZ

#### Unsubscribing from a non-subscribed key 2

The client (erroneously) unsubscribes from the same key twice in the same command.
The server is free to include the key being unsubscribed from in the `771 RPL_METADATAUNSUBOK` numeric once or twice.

Once:

    C: METADATA * SUBS
    S: :irc.example.com BATCH +byuvs metadata-subs
    S: @batch=byuvs :irc.example.com 772 modernclient website
    S: :irc.example.com BATCH -byuvs
    C: METADATA * UNSUB website website
    S: :irc.example.com 772 modernclient website

Twice:

    C: METADATA * SUBS
    S: :irc.example.com BATCH +byuvs metadata-subs
    S: @batch=byuvs :irc.example.com 772 modernclient website
    S: :irc.example.com BATCH -byuvs
    C: METADATA * UNSUB website website
    S: :irc.example.com 771 modernclient website website

#### Subscribing to a key which requires privileges but without privileges

    C: METADATA * SUB avatar secretkey website
    S: NOTE METADATA KEY_NO_PERMISSION modernclient secretkey :You do not have permission to do that.
    S: :irc.example.com 770 modernclient avatar website
    S: :irc.example.com 774 modernclient *ALL 5 :Please run SYNC later
    C: METADATA * SUBS
    S: :irc.example.com BATCH +ra20D metadata-subs
    S: @batch=ra20D :irc.example.com 772 modernclient avatar website
    S: :irc.example.com BATCH -ra20D

#### Subscribing to invalid keys and keys which requires privileges but without privileges

Because the client does not have permission to view any of the newly-added keys, the resultant `metadata` batch is empty.

    C: METADATA * SUB $invalid1 secretkey1 $invalid2 secretkey2
    S: NOTE METADATA KEY_NO_PERMISSION modernclient secretkey1 :You do not have permission to do that.
    S: WARN METADATA INVALID_KEY $invalid1 :Invalid key
    S: NOTE METADATA KEY_NO_PERMISSION modernclient secretkey2 :You do not have permission to do that.
    S: WARN METADATA INVALID_KEY $invalid2 :Invalid key
    S: :irc.example.com 770 modernclient secretkey1 secretkey2
    S: :irc.example.com BATCH +nEOfT metadata *ALL
    S: :irc.example.com BATCH -nEOfT
    C: METADATA * SUBS
    S: :irc.example.com BATCH +RInLU metadata-subs
    S: @batch=RInLU :irc.example.com 772 modernclient secretkey1 secretkey2
    S: :irc.example.com BATCH -RInLU

-----

### Setting keys and subscribing with `before-connect`

`METADATA SUB` does not generate any `metadata` batches or `774 ERR_METADATASYNCLATER` when used before connection registration has completed.

    C: CAP LS 302
    S: :metadata.test CAP * LS :batch message-tags draft/metadata-3=before-connect,max-subs=100,max-keys=100
    C: CAP REQ :batch message-tags draft/metadata-3
    S: :metadata.test CAP * ACK :batch message-tags draft/metadata-3
    C: METADATA * SUB display-name
    S: :metadata.test 770 * display-name
    C: METADATA * SET display-name :a b c
    S: :metadata.test 761 * * display-name * :a b c
    C: NICK abc
    C: USER u s e r
    C: CAP END
    S: :metadata.test 001 abc :Welcome to the MetadataTesting IRC Network abc
    S: [redacted registration burst messages]
    S: :metadata.test BATCH +1 metadata abc
    S: @batch=1 :metadata.test 761 abc abc display-name * :a b c
    S: :metadata.test BATCH -1
    S: [redacted registration burst messages]
    S: :metadata.test 376 abc :End of MOTD command
    C: METADATA * LIST
    S: :metadata.test BATCH +2 metadata abc
    S: @batch=2 :metadata.test 761 abc abc display-name * :a b c
    S: :metadata.test BATCH -2
    C: METADATA * SUBS
    S: :metadata.test BATCH +3 metadata-subs
    S: @batch=3 :metadata.test 772 abc display-name
    S: :metadata.test BATCH -3

### Differences between this specification and `metadata-notify`

*This section is non-normative.*

This specification replaces an earlier deprecated `metadata-notify` specification, by adding the following incompatible changes:

* The `metadata-notify` cap subscribed you to all keys. With the addition of explicit [`SUB`](#metadata-sub) and [`UNSUB`](#metadata-unsub) subcommands, this is no longer the case.
* Rate limiting protocol mechanics.
* Support for delayed synchronization and `METADATA SYNC`.
* Moved `ERR_*` replies to Standard Replies format.
* The `766 RPL_KEYNOTSET` numeric replaces the `766 ERR_NOMATCHINGKEY` numeric in previous versions. It is no longer an error response and has a different meaning.
* The numerics `770 RPL_METADATASUBOK`, `771 RPL_METADATAUNSUBOK`, and `772 RPL_METADATASUBS` changed from using a space separated final parameter `:<Key 1> [<Key 2> ... [<Key n>]]` to a list of individual parameters `<Key1> [<Key2> ... [<Key n>]]`.
* Use of batch instead of `762 RPL_METADATAEND` in situations where more than one `761 RPL_KEYVALUE` is sent.
* Non-standard keys should now use a vendor prefix.

## Errata

* A previous version of this specification sent `761 RPL_KEYVALUE` as the response to `METADATA CLEAR`. This was changed to `766 RPL_KEYNOTSET` to simplify client processsing.
* The `METADATA` server message was removed; `761 RPL_KEYVALUE` and `766 RPL_KEYNOTSET` are used in all cases to advertise key values or the lack thereof.
* The `SUBCOMMAND_INVALID` standard reply code has been removed. The more generic `INVALID_PARAMS` code should be used in its place.
* The `KEY_NOT_SET` standard reply code has been removed. With its removal, `METADATA SET` is fully idempotent with regards to its replies.
* The `TOO_MANY_SUBS` standard reply code has been removed. The `LIMIT_REACHED` standard reply code should be used in its place.
* The `LIMIT_REACHED` standard reply code gained a `<Command>` parameter to indicate which subcommand the limit is associated with.
* The `VALUE_INVALID` standard reply code has been renamed to `INVALID_VALUE` for better consistency between this specification and other specifications. It additionally now takes a `<Key>` parameter.
* The `KEY_INVALID` standard reply code has been renamed to `INVALID_KEY` for better consistency between this specification and other specifications.
* The `RATE_LIMITED` standard reply code was replaced with the `775 ERR_METADATARATELIMIT` numeric and is now specified for every subcommand.
* The `INVALID_PARAMS` standard reply code has been added for all other errors with `METADATA` command parameters.
* Standard reply codes have been adjusted to use `WARN` and `NOTE` when and where appropriate. `FAIL` is reserved for when the entire command cannot proceed due to an error. `WARN` is used when a portion of a command cannot proceed but other parts may still proceed. `NOTE` is used when a portion of a command is fully successful but needs ancillary data.
* The `*ALL` target has been specified for `METADATA SYNC` to synchronize all visible targets.
* Servers sending either a `metadata` batch or `774 ERR_METADATASYNCLATER` upon `JOIN`, `730 RPL_MONONLINE`, or when a client subscribes to a new key post-registration is no longer optional.
* Clients removing either the metadata or batch CAPs from themselves are now automatically unsubscribed from all keys.
* The `<RetryAfter>` parameter of `774 ERR_METADATASYNCLATER` and `775 ERR_METADATARATELIMIT` is no longer optional and must be a non-negative integer.
* A `max-key-bytes` token was added to the CAP value to limit the maximum number of bytes a metadata key may consume.
* The `before-connect` token in the CAP value list was clarified to not have its own value and that clients must ignore any value it carries.
* CAP value tokens which specify maximum lengths or numbers of entries (i.e. `max-key-bytes`, `max-value-bytes`, `max-keys`, and `max-subs`) are clarified to be unlimited if those tokens are missing from the `CAP LS 302` response.
* Empty strings are no longer valid metadata values; setting a metadata key to an empty string will now remove it (in addition to removal by omitting the value parameter entirely).
