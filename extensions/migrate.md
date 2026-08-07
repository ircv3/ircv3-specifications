# IRCv3 `migrate` Extension

*Copyright &copy; 2016 Alex Iadicicco <https://github.com/aji>*

*Copyright &copy; 2016 Justin Kaufman <https://github.com/akaritakai>*

*Unlimited redistribution and modification of this document is allowed provided
that the above copyright notice and this permission notice remains intact*

This document describes the `migrate` IRCv3 client capability and associated
protocol, which allows servers to migrate client sessions between servers, with
appropriate client cooperation.

Clients that support the migration protocol outlined in this document should
register the `migrate` client capability. If the server supports client
migrations (even if the network is not in a configuration to support
migrations), it should accept the capability.

# Protocol overview

From the client's perspective, a migration works as follows. The details of this
protocol are described later in this section.

1. The server sends `MIGRATE START` to indicate that it wants to migrate the
   client to a different server. Included in this message are a *resume token*
   and *confirm token* that will be used when connecting to the new server.

2. The client confirms the migration request with `MIGRATE OK`. After sending
   `MIGRATE OK` the client should stop sending normal IRC traffic, as it will
   not be processed. If for some reason the client does not wish to migrate,
   then the migration fails. After receiving `MIGRATE OK`, the server can begin
   coordinating the migration with the new server.

3. After the server has finished coordinating the migration with the new server,
   it sends `MIGRATE PROCEED` and closes the connection. If the connection is
   closed before receiving `MIGRATE PROCEED`, the migration fails.

4. On connecting to the new server, instead of using the normal connection setup
   protocol (`NICK`, `CAP`, etc.) the client sends a `MIGRATE RESUME` message
   containing this migration's resume token. Refer to later documentation for
   the exact semantics of this message.

5. The new server sends `MIGRATE CONFIRM`, containing this migration's confirm
   token. If the confirm token does not match, the migration fails.

6. The client sends `MIGRATE END`, and can start using the connection as it
   would have before sending `MIGRATE OK`.

Roughly, this looks as follows:

    s1 <- PRIVMSG #chan :message 1
    s1 -> :s1 MIGRATE START s2 +6697 abc def :migrate now
    s1 <- PRIVMSG #chan :message 2
    s1 <- MIGRATE OK abc
    s1 -> :s1 MIGRATE PROCEED abc
      (client disconnects from s1, connects to s2:6697 using TLS)
    s2 <- MIGRATE RESUME abc
    s2 -> :s2 MIGRATE CONFIRM def
    s2 <- MIGRATE END
    s2 <- PRIVMSG #chan :message 3

Other users in `#chan` will see messages 1 2 and 3 as if the client's connection
was left untouched.

## Client-initiated migrations

The previous example describes a migration initiated by a server, but clients
can request migrations as well using the `MIGRATE REQUEST` message. This message
contains connection details for a server the client wishes to migrate to. If the
server is willing to perform the migration, then it starts with a
`MIGRATE START` message using the normal semantics of a server-initiated
migration.

## Failures

Migrations can fail for a number of reasons:

* The client does not wish to migrate.

* The client does not send `MIGRATE OK` in a timely fashion.

* The server does not send `MIGRATE PROCEED` in a timely fashion.

* The client can't connect to the new server.

* The new server does not understand the `MIGRATE` message.

* The new server does not recognize the client's resume token.

* The new server presents an incorrect confirm token.

If at any point the client is not sure how to proceed, the client should close
the connection immediately and discard all internal state, in the same manner as
an `ERROR` message or other server-initiated disconnection.

## Timeouts

Migrations should have an associated timeout. We recommend a timeout similar to
a ping timeout, e.g. a few minutes. This way, if the client fails the migration,
the effect is similar to a connection lost for networking reasons.

## Server-to-server considerations

The following protocol is suggested for server-to-server protocols that are
based on a spanning tree. This protocol is simply advisory and is not part of
the `migrate` specification:

1. The old server (**OLD**) initiates a migration to a new server (**NEW**) by
   sending `MIGRATE START` to the client. (Servers may choose to perform a
   commitment-free negotiation of a possible migration before this step, to
   reduce the chance of the migration failing.)

2. The client accepts the migration with `MIGRATE OK`. In a single message,
   **OLD** broadcasts a "flip" message that indicates **NEW** is now responsible
   for the client. Servers receiving this message should immediately start
   routing messages for the migrated user to **NEW**.

  * The "flip" message will propagate outward from **OLD**, moving through the
    tree toward **NEW**. If a server along the path between **OLD** and **NEW**
    (including **NEW**) has outdated information and incorrectly routes a
    message toward **OLD**, the message will eventually reach a server **X**
    that has seen the "flip" message and be routed toward **NEW**. Since **X**
    has already propagated the "flip", then the re-routed message will be seen
    *after* the "flip" by each server along the path to **NEW**.

  * If **NEW** will be buffering output as bytes, then **NEW** should have all
    the information it needs to correctly format messages by the time it
    receives the "flip" message. If this information is small, it could be
    included in the "flip" itself. If this information is larger, or could
    become larger, it should be sent before the "flip" as separate messages. If
    this information can be changed externally, a smarter buffering technique
    will be needed.

  * Note that, after **OLD** has sent the "flip" message, but before the client
    has resumed its session on **NEW**, changes to client state can only come
    from other servers since **OLD** is no longer processing messages from the
    client. This fact can make it easier to reason about the migration process.

3. **OLD** sends any data it has about the client to **NEW** that **NEW** does
   not already have, such as client capabilities. During this time, **NEW** is
   receiving messages intended for the client. These messages should be
   processed by **NEW** as if the client was connected, and any output should be
   buffered.

  * Note that **NEW** already has the vast majority of information it needs
    about the client, such as nickname, user modes, services account name,
    channel membership and status, away status, and so on. Very little data is
    truly local only to **OLD**.

  * None of the data transferred during this step or received from other servers
    should affect buffering. If this is not the case, a smarter buffering
    technique is necessary so that formatting can be delayed.

4. Once all the data has has been transferred, **OLD** notifies the client with
   `MIGRATE PROCEED` and can close the connection.

5. After some time, the client will connect to **NEW** and request to resume the
   session. After the client sends `MIGRATE END`, **NEW** drains its buffer.

If any internal (non-leaf) servers do not understand the "flip" message, then
migration is not possible. A leaf only needs to identify if a client is local or
not, while internal servers take part in routing messages through the tree.
Since leaves can become internal servers through regular routing rules, then all
servers must be able to process the "flip" message for migration to be possible.

# Protocol details

## REQUEST

Clients can send `MIGRATE REQUEST` to suggest a migration.

    MIGRATE REQUEST cactus.desert :Free pizza

The parameters are

1. `REQUEST`, indicating the `MIGRATE REQUEST` operation

2. The network-internal name of the server the client wishes to be migrated to.
  Depending on the IRC network's configuration, this could be different from the
  address the client ultimately uses to connect to the new server.

3. An optional reason message.

Servers receiving this message can choose to migrate the client to the requested
server, a different server, or to reject the migration. There is no protocol for
acknowledging the request: the server simply starts a migration if the request
is accepted.

## START

The server sends a `MIGRATE START` message to the client to indicate that a
migration is starting.

    :saguaro.desert MIGRATE START cactus.desert-network.com +6697 5AEAD4D0
    9DD9F7F5 :Server going down for maintenance

The parameters are

1. `START`, identifying the command

2. The client-reachable address of the server that the client will eventually be
   migrated to. This can be either a domain name or an IP address, but should be
   reachable by the client.

3. The port that the client should use when connecting to the new server. An
   ASCII plus character (`+`) before the port indicates that TLS should be used.

4. The *resume token*, to be used by the client when connecting to the new
   server.

5. The *confirm token* token, to be presented by the new server upon migration.

6. An optional message from the server indicating the reason for the migration.

Clients should respond with `MIGRATE OK` as soon as possible. Until `MIGRATE OK`
has been received, the server should continue processing messages from the
client normally.

## OK

The client sends `MIGRATE OK` to indicate it's prepared to start the migration
process.

    MIGRATE OK 5AEAD4D0

The parameters are

1. `OK`, identifying the operation

2. The server-assigned resume token, given by the corresponding `MIGRATE START`.

`MIGRATE OK` is the last message sent by the client to the old server. Upon
receiving this message, the old server is clear to begin the process of
migrating the client to the new server. However, the client should not attempt
to resume their session until receiving a `MIGRATE PROCEED` message.

## PROCEED

The old server sends `MIGRATE PROCEED` to indicate that the client is cleared
to connect to the new server and resume the session.

    :saguaro.desert MIGRATE PROCEED 5AEAD4D0

The parameters are

1. `PROCEED`, identifying the operation.

2. The server-assigned resume token, sent in the initial `MIGRATE START`
   message.

`MIGRATE PROCEED` is the last message sent by the old server. The old connection
is now useless and both the client and server are free to close the connection
at their leisure.

## RESUME

Client send `MIGRATE RESUME` to new servers to pick up a migrated session.

    MIGRATE RESUME 5AEAD4D0

The parameters are

1. `RESUME`, identifying the operation.

2. The resume token that was assigned to this migration in the initial
   `MIGRATE START` message.

This message should be sent as the first message when opening a connection with
the new server, in place of normal registration messages like `NICK` and `CAP`.

To ensure that the new server is prepared to accept the migration, clients
should also send normal registration commands after `MIGRATE RESUME`:

    MIGRATE RESUME 5AEAD4D0
    CAP LS
    NICK aji
    USER alex * * :Alex

If the client receives a response to anything other than `MIGRATE RESUME`, the
migration should be considered failed, and the client should discard any state
it retained for the migration. The client is free to continue connecting, or to
discard the connection and start fresh.

If the server is willing to accept the migration, it should send
`MIGRATE CONFIRM` and ignore any messages sent before the associated
`MIGRATE END` from the client.

## CONFIRM

The new server sends `MIGRATE CONFIRM` to indicate they are prepared to resume
the client's session.

    :cactus.desert MIGRATE CONFIRM 9DD9F7F5

The parameters are

* `CONFIRM`, identifying the operation

* The confirm token that was sent by the old server in the initial
  `MIGRATE START` message.

The client should confirm that the provided confirm token matches the one they
received in the initial `MIGRATE START` message. If the token does not match,
the client should consider the migration failed and disconnect immediately.
Otherwise, the client should resume their session by sending a `MIGRATE END`
message.

## END

The client sends `MIGRATE END` to end the migration process and resume their
session.

    MIGRATE END

The parameters are

* `END`, identifying the operation

After sending `MIGRATE END`, the client can begin using the new connection as it
would have used the old connection before sending `MIGRATE OK`. Any messages
sent between `MIGRATE RESUME` and `MIGRATE END` should be ignored.
