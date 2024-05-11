---
title: Search
layout: spec
work-in-progress: true
copyrights:
  -
    name: "delthas"
    email: "delthas@dille.cc"
    period: "2022"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `search` capability name. Instead, implementations SHOULD
use the `draft/search` capability name to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

This feature is intended to clients to search server-side through messages that were previously sent. Server-side search enables clients to quickly find messages rather than downloading all the history from their server.

It is *not* a goal of this specification to offer context around matching messages, as this is covered by the [`chathistory`](chathistory.md) specification.

The server as mentioned in this document may refer to either an IRC server or an IRC bouncer.

Full support for this extension requires support for the batch, server-time and message-tags capabilities. However, limited functionality is available to clients without support for these CAPs. Servers SHOULD NOT enforce that clients support all related capabilities before using the search extension.

## Architecture

### Capabilities

This specification introduces the `search` capability. This capability advertises to clients that the `SEARCH` command is available.

The `search` capability MUST be negotiated.

### Batch type

This specification introduces the `search` batch type. This batch type is used when listing matching messages of a `SEARCH` command.

### Messages

This specification introduces the `SEARCH` commmand.

## Descriptiob

Clients can request a message search by sending the `SEARCH` command to the server.

This command has the following general syntax:

    SEARCH <attributes>

The `attributes` parameter is a dictionary of attributes keys and values, formatted according to the [`message-tags`](message-tags.md) specification (without the leading `@`).

If the batch capability was negotiated, the server MUST reply to a successful SEARCH command using a batch with batch type `search`. If no content exists to return, the server SHOULD return an empty batch in order to avoid the client waiting for a reply.

The server then replies with a batch of batch type `search` containing messages matching all the specified match attributes. These messages MUST be `PRIVMSG` or `NOTICE` messages.

### Returned message notes

The order of returned messages within the batch is implementation-defined, but SHOULD be ascending time order or some approximation thereof, regardless of the subcommand used. The server-time tag on each message SHOULD be the time at which the message was received by the IRC server. When provided, the msgid tag that identifies each individual message in a response MUST be the msgid tag as originally sent by the IRC server.

Servers SHOULD provide clients with a consistent message order that is valid across the lifetime of a single connection, and which determinately orders any two messages (even if they share a timestamp). This order SHOULD coincide with the order in which messages are returned within a response batch. It need not coincide with the delivery order of messages when they were relayed on any particular server.

### Errors and Warnings

Errors are returned using the standard replies syntax.

If the selectors were invalid, the `INVALID_PARAMS` error code SHOULD be returned.

    FAIL SEARCH INVALID_PARAMS [invalid_parameters] :Invalid parameters

If the search cannot be run due to an internal error, the `INTERNAL_ERROR` error code SHOULD be returned.

    FAIL SEARCH INTERNAL_ERROR [extra_context] :The search could not be run

### Standard search attributes

Servers MUST recognise the following attributes.

The following match attributes are considered a match when:
* `in`: the message was sent to this target (channel or user).
* `from`: the message was sent with this nick.
* `after`: the message was sent at or after this time (same format as the [`server-time`](server-time.md) specification).
* `before`: the message was sent at or before this time (same format as the [`server-time`](server-time.md) specification).
* `text`: the message text matches the specified text. The actual algorithm used for matching the text is implementation defined.

If `after` is specified, messages SHOULD be searched from that time. Otherwise, messages SHOULD be searched from the `before` time, which defaults to the current server time.

Additionally, the following attributes MUST be recognized:
* `limit`: a number representing an upper bound on the count of messages to return. The server MAY return fewer messages than this number.

### Examples

Searching messages sent by `jackie` in `#chan`
~~~~
[c] SEARCH from=jackie;in=#chan
[s] :irc.host BATCH +ID search
[s] @batch=ID;msgid=1234;time=2019-01-04T14:33:26.123Z :jackie!indent@host PRIVMSG #chan :Be what you want
[s] @batch=ID;msgid=1234;time=2019-01-04T14:35:26.123Z :jackie!indent@host PRIVMSG #chan :Want what you be
[s] :irc.host BATCH -ID
~~~~

Searching messages matching the text `fast` in `#chan`, returning up to 2 messages
~~~~
[c] SEARCH text=fast;in=#chan;limit=2
[s] :irc.host BATCH +ID search
[s] @batch=ID;msgid=1234;time=2019-01-04T14:33:26.123Z :bill!indent@host PRIVMSG #chan :That was fast!
[s] @batch=ID;msgid=1234;time=2019-01-04T14:35:26.123Z :jackie!indent@host PRIVMSG #chan :Fasting is hard.
[s] :irc.host BATCH -ID
~~~~

Searching messages when none match
~~~~
[c] SEARCH before=2010-01-01T00:00:00.000Z;in=#chan
[s] :irc.host BATCH +ID search
[s] :irc.host BATCH -ID
~~~~

## Implementation Considerations

Server implementations may use various algorithms for matching messages against the specified `text`. Some implementation may choose to match by substrings, by whole words, or by other algorithms such as what is offered by their database (e.g. SQLite full-text search). The comparison may be case-insensitive or case-sensitive.

## Security Considerations

Processing logs can be slow. Servers offering this feature should implement a timeout on their total request time.
