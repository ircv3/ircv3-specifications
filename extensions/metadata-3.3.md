---
title: IRCv3.3 Metadata
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Attila Molnar"
    period: "2015-2016"
    email: "attilamolnar@hush.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `metadata-notify-2` capability name. Instead, implementations SHOULD
use the `draft/metadata-notify-2` capability name to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Description

This specification extends the existing [metadata v3.2 specification](http://ircv3.net/specs/core/metadata-3.2.html).

## Metadata notify v2

This document describes the `draft/metadata-notify-2` capability.
In contrast to `metadata-notify`, where clients get METADATA events for all
keys, clients using the new capability need to explicitly subscribe to metadata
keys they are interested in.

### Capability value

The value of the metadata notify capability introduced in this document MUST
have the following format:

    [<anything>,]maxsub=<N>[,<anything>]`

`<N>` is an integer indicating the maximum number of keys a client is allowed
to be subscribed to.

## Mechanics

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

## Subcommands

Clients who negotiated the extension described in this document can use the
following new subcommands.

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

## Numerics

| No. | Label                       | Format                            |
| --- | --------------------------- | ----------------------------------|
| 775 | `RPL_METADATASUBOK`         | `:<Key1> [<Key2> ...]`            |
| 776 | `RPL_METADATAUNSUBOK`       | `:<Key1> [<Key2> ...]`            |
| 777 | `RPL_METADATASUBS`          | `:<Key1> [<Key2> ...]`            |
| 778 | `ERR_METADATATOOMANYSUBS`   | `<Key>`                           |

### RPL_METADATASUBS

The order of the keys is undefined.

### ERR_METADATATOOMANYSUBS

The <key> parameter of this numeric is the first key that the client was not
subscribed to.

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

## Examples

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
    S: CAP * LS :userhost-in-names draft/metadata-notify-2=foo,maxsub=50,bar multi-prefix

### Capability value in `CAP LS` 2

    C: CAP LS 302
    S: CAP * LS :draft/metadata-notify-2=maxsub=25 multi-prefix invite-notify
