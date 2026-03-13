---
title: User query
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Simon Ser"
    period: "2025"
    email: "contact@emersion.fr"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `user-query` CAP name. Instead, implementations SHOULD use the `draft/user-query` CAP name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use unprefixed CAP names.

## Description

This document describes the format of the `user-query` extension. This extension enables several clients of the same user connected to a server or bouncer to tell each other about which user queries (conversations between two users) are currently opened.

Some servers or bouncers remember the channels joined by a user and restore them after connection registration. This can be used by clients to synchronize channels, but doesn't work for user queries. If another client opens or closes a user query, there is no way for other clients to become aware of this action. This extension fills this gap.

The user queries opened by a user MUST NOT be disclosed to other users by the server without an explicit opt-in.

## Implementation

The `user-query` extension uses the `draft/user-query` capability, introduces a new `USERQUERY` command and a `draft/user-query-list` batch type.

The `draft/user-query` capability MUST be negotiated before using the `USERQUERY` command, and affects which messages are sent by the server as specified below.

### `USERQUERY LIST` sub-command

The `USERQUERY LIST` sub-command can be sent by clients to request a list of all opened user queries.

This sub-command has the following client syntax:

    USERQUERY LIST

The server MUST reply to a successful `USERQUERY LIST` sub-command with a `draft/user-query-list` batch if the `batch` capability has been negotiated. The batch MUST contain any number of `USERQUERY LIST` replies with the following server syntax:

    USERQUERY LIST <nick>

The `nick` parameter specifies a single nickname.

### `USERQUERY OPEN` sub-command

The `USERQUERY OPEN` sub-command can be sent by both clients and servers.

This sub-command has the following general syntax:

    USERQUERY OPEN <nick>

The `nick` parameter specifies a single nickname.

When sent from a client, this sub-command signals to the server that the user has opened a query with the specified user. The server MUST reply to a successful `USERQUERY OPEN` client set command using a `USERQUERY OPEN` server sub-command, or using an error message.

When sent from a server, the `USERQUERY OPEN` command signals to the client that the user has a query opened with the specified user. If a `PRIVMSG` is sent to the user while no user query is opened, the server automatically opens a user query before dispatching the message. Servers SHOULD broadcast the `USERQUERY OPEN` notification to all clients connected under the same account.

TODO: interactions with MONITOR?

### `USERQUERY CLOSE` sub-command

The `USERQUERY CLOSE` sub-command can be sent by both clients and servers.

This sub-command has the following general syntax:

    USERQUERY CLOSE <nick>

This command has the same syntax and behavior as `USERQUERY OPEN`, but indicates that the user has closed a user query.

### Errors and Warnings

Errors are returned using the standard replies syntax.

If the server receives a syntactically invalid `USERQUERY` command, e.g., an unknown subcommand, missing parameters, excess parameters, parameters that cannot be parsed, or invalid nickname, the `INVALID_PARAMS` error code SHOULD be returned:

    FAIL USERQUERY <subcommand> INVALID_PARAMS [invalid_parameters] :Invalid parameters

If `USERQUERY OPEN` was sent while the user query was already opened, the `ALREADY_OPENED` error code MUST be returned. If `USERQUERY CLOSE` was sent while the user query was already closed, the `ALREADY_CLOSED` error code MUST be returned.

    FAIL USERQUERY OPEN ALREADY_OPENED <nick> :This user query was already opened
    FAIL USERQUERY CLOSE ALREADY_CLOSED <nick> :This user query was already closed

If the user query cannot be opened or closed due to an error, the `INTERNAL_ERROR` error code SHOULD be returned.

    FAIL USERQUERY <subcommand> INTERNAL_ERROR <nick> :The user query could not be opened or closed

### Implementation considerations

Servers SHOULD automatically close old user queries to prevent the list of user queries from growing unbounded. For instance, this can be achieved by closing old user queries when the number of opened user queries reaches a limit, or closing user queries after an expiration timeout.

### Examples

Querying the list of opened user queries:

    C: USERQUERY LIST
    S: BATCH +foo draft/user-query-list
    S: @batch=foo USERQUERY LIST gustave
    S: @batch=foo USERQUERY LIST aline
    S: BATCH -foo

Opening a new user query:

    C: USERQUERY OPEN sophie
    S: :irc.example.org USERQUERY OPEN sophie

Closing an existing user query:

    C: USERQUERY CLOSE sophie
    S: :irc.example.org USERQUERY CLOSE sophie

The server automatically opens a user query when receiving a message:

    S: :irc.example.org USERQUERY OPEN sophie
    S: :sophie!so@example.com PRIVMSG yannick :Hi there!
