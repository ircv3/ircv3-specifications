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

The `user-query` extension uses the `draft/user-query` capability and introduces two new commands, `QUERYUSER` and `UNQUERYUSER`.

The `draft/user-query` capability MAY be negotiated, and affects which messages are sent by the server as specified below.

### `QUERYUSER` command

The `QUERYUSER` command can be sent by both clients and servers.

This command has the following general syntax:

    QUERYUSER <nick>

The `nick` parameter specifies a single nickname.

When sent from a client, this command signals to the server that the user has opened a query with the specified user. The server MUST reply to a successful `QUERYUSER` client set command using a `QUERYUSER` server command, or using an error message.

When sent from a server, the `QUERYUSER` command signals to the client that the user has a query opened with the specified user. This command MAY be sent by the server after connection registration, alongside the `JOIN` burst. If a `PRIVMSG` is sent to the user while no user query is opened, the server automatically opens a user query before dispatching the message.

TODO: interactions with MONITOR?

### `UNQUERYUSER` command

The `UNQUERYUSER` command can be sent by both clients and servers.

This command has the following general syntax:

    UNQUERYUSER <nick>

This command has the same syntax and behavior as `QUERYUSER`, but indicates that the user has closed a user query.

#### Errors and Warnings

Errors are returned using the standard replies syntax.

If the server receives a `QUERYUSER` or `UNQUERYUSER` command with missing parameters, the `NEED_MORE_PARAMS` error code MUST be returned.

    FAIL QUERYUSER NEED_MORE_PARAMS :Missing parameters

If the nickname was invalid, the `INVALID_PARAMS` error code SHOULD be returned.

    FAIL QUERYUSER INVALID_PARAMS [invalid_parameters] :Invalid parameters

If `QUERYUSER` was sent while the user query was already opened, the `ALREADY_OPENED` error code MUST be returned. If `UNQUERYUSER` was sent while the user query was already closed, the `ALREADY_CLOSED` error code MUST be returned.

    FAIL QUERYUSER ALREADY_OPENED <nick> :This user query was already opened

If the user query cannot be opened or closed due to an error, the `INTERNAL_ERROR` error code SHOULD be returned.

    FAIL QUERYUSER INTERNAL_ERROR <nick> :The user query could not be opened

### Implementation considerations

Servers SHOULD automatically close old user queries to prevent the list of user queries from growing unbounded. For instance, this can be achieved by closing old user queries when the number of opened user queries reaches a limit, or closing user queries after an expiration timeout.

### Examples

Sending the list of opened user queries after connection registration:

    S: 422 yannick :No message of the day
    S: :yannick!yan@example.org JOIN #ircv3
    S: :yannick!yan@example.org QUERYUSER gustave

Opening a new user query:

    C: QUERYUSER sophie
    S: :yannick!yan@example.org QUERYUSER sophie

Closing an existing user query:

    C: UNQUERYUSER sophie
    S: :yannick!yan@example.org UNQUERYUSER sophie

The server automatically opens a user query when receiving a message:

    S: :yannick!yan@example.org QUERYUSER sophie
    S: :sophie!so@example.com PRIVMSG yannick :Hi there!
