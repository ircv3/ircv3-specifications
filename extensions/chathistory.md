---
title: IRCv3 chathistory extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Evan Magaliff"
    period: "2017"
    email: "evan@muffinmedic.net"
  -
    name: "Darren Whitlen"
    period: "2018-2019"
    email: "darren@kiwiirc.com"
---
## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `chathistory` or `event-playback` CAP names. Instead, implementations SHOULD use the `draft/chathistory` and `draft/event-playback` CAP names to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use unprefixed CAP names.

The `chathistory` batch type is already ratified and SHOULD be used unprefixed.

## Description
This document describes the format of the `chathistory` extension. This enables clients to request messages that were previously sent if they are still available on the server.

The server as mentioned in this document may refer to either an IRC server or an IRC bouncer.

## Implementation
The `chathistory` extension uses the [chathistory][batch/chathistory] batch type and introduces a client command, `chathistory`.

Full support for this extension requires support for the [`batch`][batch], [`server-time`][server-time] and [`message-tags`][message-tags] capabilities. However, limited functionality is available without support for these CAPs. Servers SHOULD NOT enforce that clients support all related capabilities before using the `chathistory` extension.

The `draft/chathistory` capability MUST be negotiated. This allows the server and client to act differently when delivering message history on connection.

An ISUPPORT token MUST be sent to the client to state the maximum number of messages a client can request in a single command, represented by an integer. `CHATHISTORY=50`. If `0`, the client SHOULD assume that there is no maximum number of messages.

The `draft/event-playback` capability MAY be negotiated. This allows the client to signal that it is capable of receiving and correctly processing lines that would normally produce a local state change (such as `JOIN` or `MODE`) in its history batches.

### `CHATHISTORY` Command
`CHATHISTORY` content can be requested by the client by sending the `CHATHISTORY` command to the server. A `batch` MUST be returned by the server. If no content exists to return, an empty batch SHOULD be returned to avoid the client waiting for a reply and to indicate that no content is available.

The `chathistory` command uses the following general syntax structure:

    CHATHISTORY <subcommand> <target> <timestamp | msgid> <limit>

The `target` parameter specifies a single buffer (channel or nickname) from which history SHOULD be retrieved. Any `timestamp` values or parameters MUST be in the format of `YYYY-MM-DDThh:mm:ss.sssZ`.

The special target `*` refers to all direct messages sent to or from the current user, regardless of the other party. This allows the client to retrieve conversations with users it is not yet aware of.

If a nickname is given as the `target` then the server SHOULD include history sent between the current user and the target nickname, including outgoing messages ("self messages"), so as to give a full conversation. The server SHOULD attempt to include history involving other nicknames if either the current user or the target nickname has changed during the requested timeframe.

#### Subcommands

The following subcommands are used to describe how the server should return messages relative to the `timestamp` or `msgid` given.

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
Request up to `limit` number of messages between the given `timestamp` or `msgid` values. The returned messages MUST start from and exclude the first message selector, while finishing on and excluding the second - this may be forwards or backwards in time.

#### Returned message notes
The order of returned messages within the batch is implementation-defined, but SHOULD be ascending time order or some approximation thereof. The `server-time` tag on each message SHOULD be the time at which the message was received by the IRC server. The `msgid` tag that identifies each individual message in a response MUST be the `msgid` tag as originally sent by the IRC server.

Servers SHOULD provide clients with a consistent message order that is valid across the lifetime of a single connection, and which determinately orders any two messages (even if they share a timestamp); this will allow BEFORE, AFTER, and BETWEEN queries that use msgids for pagination to function as expected. This order SHOULD coincide with the order in which messages are returned within a response batch. It need not coincide with the delivery order of messages when they were relayed on any particular server.

If the client has not negotiated the `event-playback` capability, the server MUST NOT send any lines other than `PRIVMSG` and `NOTICE` in the reply batch. If the client has negotiated `event-playback`, the server SHOULD send additional lines relevant to the chat history, including but not limited to `TAGMSG`, `JOIN`, `PART`, `QUIT`, `MODE`, `TOPIC`, and `NICK`.

#### Errors and Warnings
Errors are returned using the standard replies syntax.

If the server receives a `CHATHISTORY` command with an unknown subcommand, the `UNKNOWN_COMMAND` error code MUST be returned.
> FAIL CHATHISTORY UNKNOWN_COMMAND the_given_command :Unknown command

If the server receives a `CHATHISTORY` command with missing parameters, the `NEED_MORE_PARAMS` error code MUST be returned.
> FAIL CHATHISTORY NEED_MORE_PARAMS the_given_command :Missing parameters

If no message history can be returned due to an error, the `MESSAGE_ERROR` error code SHOULD be returned.
> FAIL CHATHISTORY MESSAGE_ERROR the_given_command the_given_target [extra_context] :Messages could not be retrieved

### Examples

Requesting the latest conversation upon joining a channel
~~~~
[c] CHATHISTORY LATEST #channel * 50
[s] :irc.host BATCH +ID chathistory #channel
[s] @batch=ID;msgid=1234;time=2019-01-04T14:33:26.123Z :nick!ident@host PRIVMSG #channel :message
[s] @batch=ID;msgid=1235;time=2019-01-04T14:33:38.123Z :nick!ident@host NOTICE #channel :message
[s] @batch=ID;msgid=1238;time=2019-01-04T14:34:17.123Z;+client_tag=val :nick!ident@host PRIVMSG #channel :ACTION message
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
The batch type and supporting command are useful for allowing an "infinite scroll" type capability within the client. A client will, upon scrolling to the top of the active window or a manual trigger, may request `chathistory` from the server and, after receiving returned content, append it to the top of the window. Users can repeat this historic scrolling to retrieve prior history until limitations are met (see below).

Upon joining a channel, a client may request the latest messages for the buffer so that the active conversation context may be retrieved.

### Client pseudocode
A client with full support for BATCH, message IDs, and deduplication can fill in gaps in its history using the following pseudocode. `FUZZ_INTERVAL` is a constant that compensates for clock skew across the IRC network (perhaps 1 to 10 seconds):

    lower_bound = <timestamp of last message relayed to the previous session>
    lower_bound -= FUZZ_INTERVAL
    upper_bound = now() + FUZZ_INTERVAL
    retrieved_count = 0
    while retrieved_count < SANITY_LIMIT:
        messages = CHATHISTORY(BEFORE, upper_bound)
        if len(messages) == 0:
            break
        retrieved_count += len(messages)
        earliest_message = messages[0]
        display(deduplicate_messages)
        if earliest_message.timestamp < lower_bound:
            break
        upper_bound = earliest_message.msgid

A client without support for BATCH, message IDs, or deduplication can still make use of CHATHISTORY, albeit with the possibility of skipping some messages or seeing some duplicated messages. For example, on initial JOIN, the client can do the following (this implementation errs on the side of missing messages, rather than seeing duplicated messages):

    display(CHATHISTORY(BEFORE, join_message.msgid))

Infinite scroll can be implemented as:

    lower_bound_msg = <last message relayed to previous session>
    upper_bound_msg = <earliest message relayed or retrieved during current session>
    messages = CHATHISTORY(BETWEEN, upper_bound_msg.msgid, lower_bound_msg.msgid)
    display(messages)
    upper_bound_msg = messages[0]

To fill in gaps in a client without deduplication support, iterate this infinite scrolling operation until the BETWEEN query returns no results (or until a sane limit of retrieved messages is reached).

## Implementation Considerations

In the typical IRC network, there is no well-defined global linear ordering of messages, since different linked servers may see messages in different orders. Furthermore, due to clock skew between servers and between server and client, messages may be relayed, stored, and replayed in an order that differs from the timestamp order. Within the recommended constraints on message ordering described above, implementations may want to make different tradeoffs between simplicity, consistency, and being sure of receiving all relevant history messages.

## Security Considerations

Secure identification of users and clients MUST exist in order to ensure that users cannot obtain history they are not authorised to view. Use of account names, internal account identifiers, or certificate fingerprints SHOULD be strongly considered when matching content to users. If a client requests content for a target that they do not have permission for, eg. a channel they are banned from, an empty batch SHOULD be returned as if no content exists.

While an ISUPPORT token value of `0` may be used to indicate no message limit exists, servers SHOULD set and enforce a reasonable maximum and properly throttle `CHATHISTORY` commands to prevent abuse.
