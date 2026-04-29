---
title: Persistence
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Shivaram Lingamneni"
    period: "2022"
    email: "slingamn@cs.stanford.edu"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `persistence` CAP name. Instead, implementations SHOULD use the `draft/persistence` CAP name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use unprefixed CAP names.

## Introduction

Some IRC server implementations offer a mode of operation where clients can remain present on the server even when they are disconnected: the client's nickname remains unavailable to other clients, they remain joined to their channels, and they remain visible to their channel co-participants. Typically, such implementations are bouncers (i.e., intermediaries between the client and another server), but some are full server implementations.

Despite the longstanding existence of such servers, most of the conventions for client-to-server interaction in this context remain implicit. This document describes the `persistence` extension, which aims to explicitly clarify these interactions:

* Servers can communicate to clients whether their connection is "persistent" in the above sense, so clients can adjust their behavior accordingly
* Clients can request that the server make their connection persistent or non-persistent, and the server can grant or deny the request

## Implementation

For the purposes of this specification, a "persistent" client, or a client with "persistence enabled", is a client that remains present on the server even if it has no active client-to-server connections.

The `persistence` extension uses the `draft/persistence` capability and introduces a new command, `PERSISTENCE`.

## The `PERSISTENCE` command

The `PERSISTENCE` command is the primary means of communicating persistence settings. It has three subcommands. Implementations SHOULD ignore unrecognized subcommands, to be compatible with future extensions of this specification.

### `PERSISTENCE STATUS`

The `STATUS` subcommand is sent from the server to the client:

    PERSISTENCE STATUS <client-setting> <effective-setting>

`<client-setting>` has three possible values:

* `ON`, indicating that the client has requested that persistence be enabled
* `OFF`, indicating that the client has requested that persistence be disabled
* `DEFAULT`, indicating that the client has no preference

`<effective-setting>` has two possible values:

* `ON`, indicating that persistence is in fact enabled
* `OFF`, indicating that persistence is in fact disabled

If the client is authenticated and the `draft/persistence` capability is negotiated, servers MUST send a `PERSISTENCE STATUS` command to the client as part of the registration burst, after the last `005` numeric but before either `376 RPL_ENDOFMOTD` or `422 ERR_NOMOTD`. This allows the client to adjust their behavior accordingly. If the `draft/persistence` capability is negotiated and an event occurs asynchronously that changes the persistence status (e.g. a server configuration change or `PERSISTENCE SET` from another client connection), servers SHOULD send `PERSISTENCE STATUS` to inform the client.

If persistence is enabled, the server MUST send a `JOIN` line for each channel the client is joined to, subsequent to sending either `376` or `422`.

### `PERSISTENCE GET`

The `GET` subcommand is sent from the client to the server:

    PERSISTENCE GET

The server replies with a `PERSISTENCE STATUS` command as specified above.

### `PERSISTENCE SET`

The `SET` subcommand is sent from the client to the server:

    PERSISTENCE SET <client-setting>

where `<client-setting>` takes one of the three values specified above; the server replies with a `PERSISTENCE STATUS` command as specified above.

#### Errors and Warnings

Servers SHOULD NOT allow unauthenticated clients to become persistent, since there is currently no way for the client to reconnect in that case. Unauthenticated clients attempting to issue `GET` or `SET` SHOULD receive:

    FAIL PERSISTENCE ACCOUNT_REQUIRED :An account is required

On invalid parameters, the server MUST return:

    FAIL PERSISTENCE INVALID_PARAMETERS :Invalid parameters

If persistence cannot be accessed or modified due to an error, the `INTERNAL_ERROR` error code SHOULD be returned.

    FAIL PERSISTENCE INTERNAL_ERROR :An error occurred

Finally, servers MAY accept the change in client setting while not changing the effective setting. For example, persistence may be mandatory for all users, or it may be reserved for privileged users. In this event, the server MUST NOT send a `FAIL` response, but MUST send a `PERSISTENCE STATUS` response describing the current settings. The server MAY send a `WARN` or `NOTE` response explaining why the change was refused.

### Examples

A client enables persistence on a server with opt-in persistence:
~~~~
[c] PERSISTENCE GET
[s] PERSISTENCE STATUS DEFAULT OFF
[c] PERSISTENCE SET ON
[s] PERSISTENCE STATUS ON ON
~~~~

A client attempts to enable persistence, but server policy does not allow them to become persistent:

~~~~
[c] PERSISTENCE SET ON
[s] PERSISTENCE STATUS ON OFF
~~~~

A client attempts to disable persistence, but server policy requires that they remain persistent:

~~~~
[c] PERSISTENCE GET
[s] PERSISTENCE STATUS DEFAULT ON
[c] PERSISTENCE SET OFF
[s] PERSISTENCE STATUS OFF ON
~~~~

A client negotiates the `draft/persistence` capability and sees that persistence is enabled for them:

~~~~
[c] CAP REQ :draft/persistence sasl
[s] :irc.example.com CAP * ACK :draft/persistence sasl
[c] NICK testuser
[c] USER u s e r
[c] AUTHENTICATE PLAIN
[s] AUTHENTICATE +
[c] AUTHENTICATE amlsbGVzAGppbGxlcwBzZXNhbWU=
[s] :irc.example.com 900 * * jilles :You are now logged in as jilles
[s] :irc.example.com 903 * :Authentication successful
[c] CAP END
[s] :irc.example.com 001 testuser :Welcome to the example IRC network testuser
[...]
[s] :irc.example.com 005 testuser :TOPICLEN=390 UTF8ONLY WHOX draft/CHATHISTORY=1000 :are supported by this server
[...]
[s] :irc.example.com PERSISTENCE ON ON
[s] :irc.example.com 375 testuser :- irc.example.com Message of the day -
[s] :irc.example.com 372 testuser :- welcome to irc.example.com, where we offer persistent connections
[s] :irc.example.com 376 testuser :End of MOTD command
~~~~

## Implementation Considerations

Clients observing that their connection is persistent SHOULD NOT attempt to automatically rejoin channels; rather, they should wait for the server to send JOIN lines for channels that the persistent presence is already a member of. This prevents such clients from rejoining channels that were PART'ed by a different client associated with the same presence.

## Recommendations

This section is non-normative.

Persistence carries a high risk of abuse, including denial-of-service attacks against servers. Server implementations and server operators should institute safeguards against this (for example, requiring verification during account registration).

Servers may implement implementation-defined or configurable conditions for removing persistent clients from the server (for example, an inactivity timeout on the order of days or months).

Conventional IRC bouncers can implement this specification by adding support for the `PERSISTENCE` command and support for storing the user's preferred setting, then communicating that regardless of the user's setting, persistence is still enabled (e.g. `PERSISTENCE OFF ON`). This communicates to the client that any preference to become non-persistent has been received but will not be honored.

There are multiple possibilities for client UIs. Some clients may wish to enable persistence by default (allowing the user to disable this either globally or per-network). They can request the `draft/persistence` capability, then follow the following algorithm:

* On observing `PERSISTENCE STATUS DEFAULT OFF`, send `PERSISTENCE SET ON`
* On observing `PERSISTENCE STATUS ON ON`, persistence has been enabled successfully and this can be displayed to the end user
* On observing `PERSISTENCE STATUS ON OFF`, report to the end user that it is not possible to enable persistence
* On observing `PERSISTENCE STATUS OFF OFF`, report to the end user that persistence has been disabled by another client

Alternately, clients may wish to report the persistence status to the user, then provide a setting to change it.
