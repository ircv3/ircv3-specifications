---
title: "`chathistory` Extension"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "MuffinMedic"
    period: "2017-2018"
  -
    name: "Darren Whitlen"
    period: "2018-2020"
    email: "darren@kiwiirc.com"
  -
    name: "Shivaram Lingamneni"
    period: "2020"
    email: "slingamn@cs.stanford.edu"
  -
    name: "Val Lorentz"
    period: "2022"
    email: "progval+ircv3@progval.net"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `chathistory` or `event-playback` CAP names. Instead, implementations SHOULD use the `draft/chathistory` and `draft/event-playback` CAP names to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use unprefixed CAP names.

The `chathistory` batch type is already ratified and SHOULD be used unprefixed.

## Introduction
This document describes the format of the `chathistory` extension. This enables clients to request messages that were previously sent if they are still available on the server.

The server as mentioned in this document may refer to either an IRC server or an IRC bouncer.

## Implementation
The `chathistory` extension uses the [chathistory][batch/chathistory] batch type and introduces a new client command, `CHATHISTORY`.

Full support for this extension requires support for the [`batch`][batch], [`server-time`][server-time] and [`message-tags`][message-tags] capabilities. However, limited functionality is available to clients without support for these CAPs. Servers SHOULD NOT enforce that clients support all related capabilities before using the `chathistory` extension. Meanwhile, bouncers implementing the server side of this specification may not be able to provide message IDs (when they are mediating for a server that does not support the `message-tags` capability). Therefore, clients with full support for `message-tags` MAY wish to implement fallback logic that relies only on `server-time`.

The `draft/chathistory` capability MUST be negotiated. This allows the server and client to behave differently when history is available and can be queried. In particular, if a client requests this capability, the server (or bouncer) SHOULD NOT play back any history that would otherwise be sent automatically.

The `draft/event-playback` capability MAY be negotiated. This allows the client to signal that it is capable of receiving and correctly processing lines that would normally produce a local state change (such as `JOIN` or `MODE`) in its history batches.

### ISUPPORT tokens

This specification defines two ISUPPORT tokens.

`CHATHISTORY` MUST be sent to the client to state the maximum number of messages a client can request in a single command, e.g., `CHATHISTORY=50`. If `0`, the client SHOULD assume that there is no maximum number of messages.

`MSGREFTYPES` SHOULD be sent, with a comma-separated list of types of message references they support as parameter to `CHATHISTORY` commands, in order of decreasing preference. Clients MUST ignore any type they do not support. The currently defined list of types is:

* `timestamp`, to reference messages by their timestamp
* `msgid`, to reference them by their `msgid`

If `MSGREFTYPES` is missing, clients SHOULD assume the server supports both and has no preference.


### `CHATHISTORY` Command
The client can request message history content by sending the `CHATHISTORY` command to the server. This command has the following general syntax:

    CHATHISTORY <subcommand> <target> <timestamp | msgid> [<timestamp | msgid>] <limit>
    CHATHISTORY TARGETS <timestamp> <timestamp> <limit>

The `target` parameter specifies a single buffer (channel or nickname) from which history is to be retrieved. If a nickname is given as the `target` then the server SHOULD include history sent between the current user and the target nickname, including outgoing messages ("self messages"). The server SHOULD attempt to include history involving other nicknames if either the current user or the target nickname has changed during the requested timeframe.

A `timestamp` parameter MUST have the format `timestamp=YYYY-MM-DDThh:mm:ss.sssZ`, as in the [server-time][server-time] extension. A `msgid` parameter MUST have the format `msgid=foobar`, as in the [message-ids][message-ids] extension.

If the `batch` capability was negotiated, the server MUST reply to a successful `CHATHISTORY` command using a [`batch`][batch]. For subcommands that return message history (i.e. all subcommands other than `TARGETS`), the batch MUST have type `chathistory` and take a single additional parameter, the canonical name of the target being queried. For `TARGETS`, the batch MUST have type `draft/chathistory-targets`. If no content exists to return, the server SHOULD return an empty batch in order to avoid the client waiting for a reply.

If the client has not negotiated the `draft/event-playback` capability, the server MUST NOT send any lines other than `PRIVMSG` and `NOTICE` in the reply batch, unless allowed by a capability negotiated by the client. If the client has negotiated `draft/event-playback`, the server SHOULD send additional lines relevant to the chat history, including but not limited to `TAGMSG`, `JOIN`, `PART`, `QUIT`, `MODE`, `TOPIC`, and `NICK`.

#### Subcommands

The following subcommands are used to return message history relative to `timestamp` or `msgid` selection parameter(s).

#### `BEFORE`

    CHATHISTORY BEFORE <target> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ | msgid=1234> <limit>

Request up to `limit` number of messages before and excluding the given `timestamp` or `msgid`. Only one timestamp or msgid MUST be given, not both.

#### `AFTER`

    CHATHISTORY AFTER <target> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ | msgid=1234> <limit>

Request up to `limit` number of messages after and excluding the given `timestamp` or `msgid`. Only one timestamp or msgid MUST be given, not both.

#### `LATEST`

    CHATHISTORY LATEST <target> <* | timestamp=YYYY-MM-DDThh:mm:ss.sssZ | msgid=1234> <limit>

Request up to `limit` number of the most recent messages that have been sent. If a `timestamp` or `msgid` is given, the returned messages are restricted to those sent after and excluding that timestamp or msgid; if a `*` is given, no such restriction applies. If a `*` is not given, only one timestamp or msgid MUST be given, not both.

This is useful for retrieving the latest conversation when first joining a channel or opening a query buffer.

#### `AROUND`

    CHATHISTORY AROUND <target> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ | msgid=1234> <limit>

Request a number of messages before and after the `timestamp` or `msgid` with the total number of returned messages not exceeding `limit`. The implementation may decide how many messages to include before and after the selected message. Only one timestamp or msgid MUST be given, not both.

This is useful for retrieving conversation context around a single message.

#### `BETWEEN`

    CHATHISTORY BETWEEN <target> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ | msgid=1234> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ | msgid=1234> <limit>

Request up to `limit` number of messages between the given `timestamp` or `msgid` values. With respect to the limit, the returned messages MUST be counted starting from and excluding the first message selector, while finishing on and excluding the second. This may be forwards or backwards in time.

#### `TARGETS`

Unlike the other subcommands, `TARGETS` does not return message history. Instead, it lists channels the user has visible history in, together with users with which the user has exchanged direct messages, ordered by the time of the latest message in the channel history or direct message conversation (i.e. sent from or to the other user). This allows the client to discover missed direct message conversations with users it is not currently aware of.

    CHATHISTORY TARGETS <timestamp=YYYY-MM-DDThh:mm:ss.sssZ> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ> <limit>

The parameters have the same semantics as `BETWEEN`, except that they MUST be timestamps and not msgids. Returned messages have the syntax:

    CHATHISTORY TARGETS <nickname | channel name> <YYYY-MM-DDThh:mm:ss.sssZ>

where the timestamp is the time of the latest message in the channel history or direct message conversation.

#### Returned message notes
The order of returned messages within the batch is implementation-defined, but SHOULD be ascending time order or some approximation thereof, regardless of the subcommand used. The `server-time` tag on each message SHOULD be the time at which the message was received by the IRC server. The `msgid` tag that identifies each individual message in a response MUST be the `msgid` tag as originally sent by the IRC server.

Servers SHOULD provide clients with a consistent message order that is valid across the lifetime of a single connection, and which determinately orders any two messages (even if they share a timestamp); this will allow BEFORE, AFTER, and BETWEEN queries that use msgids for pagination to function as expected. This order SHOULD coincide with the order in which messages are returned within a response batch. It need not coincide with the delivery order of messages when they were relayed on any particular server.

#### `draft/chathistory-context` tag

The server may choose to return additional, related messages alongside regular chat history. Such messages MUST be tagged with the `draft/chathistory-context` tag and MUST immediately follow their parent message. Context messages MUST NOT be counted towards the message limit. Context messages are sent at the server's discretion and MAY include reacts, redacts, edits, etc.

#### Errors and Warnings
Errors are returned using the standard replies syntax.

If the server receives a syntactically invalid `CHATHISTORY` command, e.g., an unknown subcommand, missing parameters, excess parameters, or parameters that cannot be parsed, the `INVALID_PARAMS` error code SHOULD be returned:

    FAIL CHATHISTORY INVALID_PARAMS the_given_command :Unknown command
    FAIL CHATHISTORY INVALID_PARAMS the_given_command :Insufficient parameters
    FAIL CHATHISTORY INVALID_PARAMS the_given_command :Too many parameters
    FAIL CHATHISTORY INVALID_PARAMS the_given_command the_given_timestamp :Invalid timestamp

If an existing error numeric such as `ERR_NEEDMOREPARAMS` is appropriate, it MAY be used instead.

If the target does not exist or the client does not have permissions to query it, the `INVALID_TARGET` error code SHOULD be returned:

    FAIL CHATHISTORY INVALID_TARGET the_given_command the_given_target :Messages could not be retrieved

If no message history can be returned due to an error, the `MESSAGE_ERROR` error code SHOULD be returned:

    FAIL CHATHISTORY MESSAGE_ERROR the_given_command the_given_target [extra_context] :Messages could not be retrieved

If a client used a reference type (`timestamp=` or `msgid=`) the server does not support, the `INVALID_MSGREFTYPE` error code SHOULD be returned:

    FAIL CHATHISTORY INVALID_MSGREFTYPE the_given_command the_given_target [extra_context] :msgid-based history requests are not supported

### Examples

Requesting the latest conversation upon joining a channel
~~~~
[c] CHATHISTORY LATEST #channel * 50
[s] :irc.host BATCH +ID chathistory #channel
[s] @batch=ID;msgid=1234;time=2019-01-04T14:33:26.123Z :nick!ident@host PRIVMSG #channel :message
[s] @batch=ID;msgid=1235;time=2019-01-04T14:33:38.123Z :nick!ident@host NOTICE #channel :message
[s] @batch=ID;msgid=1238;time=2019-01-04T14:34:17.123Z;+client-tag=val :nick!ident@host PRIVMSG #channel :ACTION message
[s] :irc.host BATCH -ID
~~~~

Requesting further message history than our client currently has
~~~~
[c] CHATHISTORY BEFORE bob timestamp=2019-01-04T14:34:17.123Z 50
[s] :irc.host BATCH +ID chathistory bob
[s] @batch=ID;msgid=1234;time=2019-01-04T14:34:09.123Z :bob!ident@host PRIVMSG alice :hello
[s] @batch=ID;msgid=1235;time=2019-01-04T14:34:10.123Z :alice!ident@host PRIVMSG bob :hi! how are you?
[s] @batch=ID;msgid=1238;time=2019-01-04T14:34:16.123Z; :bob!ident@host PRIVMSG alice :I'm good, thank you!
[s] :irc.host BATCH -ID
~~~~

## Use Cases
Upon joining a channel, a client may request the latest messages for the channel, to retrieve the immediate context of the active conversation.

Clients can use `CHATHISTORY` to implement "infinite scroll". When the user scrolls to the top of the active window or engages a manual trigger, the client can request `CHATHISTORY` from the server and then insert the results at the top of the window. The user can repeat this action to retrieve more history, possibly until some limit is met.

### Client pseudocode
A client with full support for BATCH, message IDs, and deduplication can fill in gaps in its history using the following pseudocode. `FUZZ_INTERVAL` is a constant that compensates for clock skew across the IRC network (perhaps 1 to 10 seconds):

    lower_bound = <timestamp of last message relayed to the previous session>
    lower_bound -= FUZZ_INTERVAL
    upper_bound = None
    retrieved_count = 0
    while retrieved_count < SANITY_LIMIT:
        if upper_bound is None:
            messages = CHATHISTORY(LATEST, target, *)
        else:
            messages = CHATHISTORY(BEFORE, target, upper_bound)
        if len(messages) == 0:
            break
        retrieved_count += len(messages)
        earliest_message = messages[0]
        display(deduplicate_messages)
        if earliest_message.timestamp < lower_bound:
            break
        upper_bound = earliest_message.msgid

A client without support for BATCH, message IDs, or deduplication can still make use of CHATHISTORY, albeit with the possibility of skipping some messages or seeing some duplicated messages. For example, on initial JOIN, the client can do the following:

    display(CHATHISTORY(LATEST, target, *))

To avoid the possibility of seeing duplicated messages here (messages that were relayed after the channel join, but also appear in the CHATHISTORY LATEST output), a client could ignore messages relayed to the channel until the CHATHISTORY reply batch is complete.

Infinite scroll can be implemented as:

    lower_bound_msg = <last message relayed to previous session>
    upper_bound_msg = <earliest message relayed or retrieved during current session>
    messages = CHATHISTORY(BETWEEN, target, upper_bound_msg.msgid, lower_bound_msg.msgid)
    display(messages)
    upper_bound_msg = messages[0]

To fill in gaps in a client without deduplication support, iterate this infinite scrolling operation until the BETWEEN query returns no results (or until a sane limit of retrieved messages is reached).

To use `TARGETS` to retrieve missed direct messages on reconnection:

    upper_bound = now()
    upper_bound += FUZZ_INTERVAL
    lower_bound = <timestamp of last message relayed to the previous session>
    lower_bound -= FUZZ_INTERVAL
    targets = CHATHISTORY(TARGETS, upper_bound, lower_bound, limit)
    for target in targets:
        target_lower_bound = <timestamp of last message exchanged with target>
        CHATHISTORY(LATEST, target.name, target_lower_bound, limit)

## Implementation Considerations

In the typical IRC network, there is no well-defined global linear ordering of messages, since different linked servers may see messages in different orders. Furthermore, due to clock skew between servers and between server and client, messages may be relayed, stored, and replayed in an order that differs from the timestamp order. Within the recommended constraints on message ordering described above, implementations may want to make different tradeoffs between simplicity, consistency, and correctness (i.e., neither missing nor duplicating messages).

Client implementations should account for the possibility that history reply batches may contain nicknames (as sources or parameters) that have subsequently changed. With direct message history, servers may wish to rewrite the sources or targets of messages to correspond to the current nicknames of the two users.

## Security Considerations

Servers MUST ensure that users cannot obtain history they are not authorised to view. Servers SHOULD use secure identification mechanisms such as account names, internal account identifiers, or certificate fingerprints to match content to users.

Given conventional expectations around channel membership, servers MAY wish to disallow clients from querying the history of channels they are not joined to. If they do not, they SHOULD disallow clients from querying channels that they are banned from, or which are private.

While an ISUPPORT token value of `0` may be used to indicate no message limit exists, servers SHOULD set and enforce a reasonable maximum and properly throttle `CHATHISTORY` commands to prevent abuse.

[batch/chathistory]: ../batches/chathistory.html
[batch]: ../extensions/batch.html
[server-time]: ../extensions/server-time.html
[message-tags]: ../extensions/message-tags.html
[message-ids]: ../extensions/message-ids.html
