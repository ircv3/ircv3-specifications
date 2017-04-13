---
title: IRCv3 `resume` extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: Daniel Oakley
    period: 2017
    email: daniel@danieloaks.net
---
## Introduction

Occasionally, clients disconnect from IRC. What happens these days is that the client connects with a different nick, joins all their old channels again, waits for the old connection to time out (or manually kills it using services), and then changes back to their original nickname.

This feature intends to vastly simplify this form of reconnection, and reduce the amount of nick-switching, `JOIN` and `QUIT` notices, and general disruption that other clients see when this happens.


## Architecture

This feature is enabled using the `draft/resume` capability, and uses various new messages and numerics to convey state about reconnecting clients.

### Capabilities

The `draft/resume` capability is advertised by servers that support resuming connections. This capability is requested by clients that reconnect in this way with the `RESUME` command and have the ability to correctly parse the `RESUMED` message as described in this specification.

### Messages

There are two new messages defined by this specification:

- `RESUME`: Indicates that the client wishes to resume a previous connection.
- `RESUMED`: Indicates that a client has disconnected+reconnected to the server.

Below, we go through both of these.

#### `RESUME` Command

This command is sent before starting SASL authentication, and indicates that the client wishes to resume a previous connection to the server which may still be active. The command is sent with the following format:

    RESUME oldnick timestamp

`oldnick` is the nickname the client had when they last disconnected. `timestamp` is a timestamp which indicates either when they received the last message from the server on the old connection, or just when they disconnected from the server. This timestamp is used to indicate to other clients how long the disconnection was for, and uses the same format as the IRCv3 `server-time` extension (i.e. `YYYY-MM-DDThh:mm:ss.sssZ`, or in UTC using extended format as specified by ISO 8601:2004(E) 4.3.2).

After sending this command, the client continues connection registration as usual.

If the client's able to successfully resume their connection, the server sends them a `RESUMED` message before sending the usual registration burst (and their nickname is set to `<oldnick>`). Otherwise, the server sends the `ERR_CANNOT_RESUME` numeric.

#### `RESUMED` Message

This message is sent when a client successfully resumes a connection. For clients that have not completed connection registration, this indicates that their `RESUME` attempt was successful. For registered clients, this indicates that the given client disconnected and reconnected to the server.

This message has the following format:

    RESUMED nick user host timestamp

`nick` is the nickname of the client whose connection has been resumed. `user` and `host` are the username and hostname that the new connecting client has. The client receiving this message MUST update the username and hostname they have stored for the given client, in the same way that the `CHGHOST` message is parsed.

### Numerics

Here are the numerics that this extension defines:

| No. | Label                   | Format                                                          |
| --- | ----------------------- | --------------------------------------------------------------- |
| ??? | `ERR_CANNOT_RESUME`     | `:<server> ??? <oldnick> :Cannot resume connection, <details>`  |
| ??? | `ERR_HISTORY_TRUNCATED` | `:<server> ??? <client/channel> :Chat history may be truncated` |

`ERR_CANNOT_RESUME` is used to indicate a general failure to resume a connection for a specific reason. The purpose of this numeric is to pass along human-readable information informing the user why their connection wasn't automatically resumed such as a lack of SASL authentication and login. If a more appropriate numeric exists for conveying the issue already exists (such as `ERR_NOSUCHNICK` to represent the old nickname no longer being present on the network), then it should be used instead.

Implementations should keep in mind that the only concrete indication that a `RESUME` attempt failed is that it does not receive a `RESUMED` message during registration. Clients MUST NOT rely on `ERR_CANNOT_RESUME` or any other numeric being received to trigger their 'resume has failed' state and process.

`ERR_HISTORY_TRUNCATED` is used to indicate that chat history may be truncated for a given client/channel. It may be sent during a `chathistory` batch or otherwise during history playback. Bouncers that reconnect may also find this useful to send to their connected clients, to indicate that there may be missed history.


## Connection Registration

Upon connection, clients request the `draft/resume` capability. If they receive both this capability and the SASL capability, they can continue attempting to resume their old connection.

After sending the `NICK` and `USER` commands, clients wishing to resume an old connection send an appropriate `RESUME` command indicating the nickname they wish to take over and how long they've been disconnected for.

After this, the client performs SASL authentication to prove that the indicated old connection does belong to them. If SASL authentication completes successfully, clients are sent the `RESUMED` message and then the regular connection burst (`001`-`005` and all).

Once the connection burst has been received, the server sends the client channel `JOIN` messages as appropriate (and the usual numerics that are sent on joining a channel), and dispatches the relevant messages and numerics to other clients in related channels. The new client is also given the user modes that were active on the old client, as appropriate.

Other clients that have the reconnecting user `MONITOR`'d MUST be sent one `RPL_MONOFFLINE` numeric and one `RPL_MONONLINE` numeric indicating that the user reconnected.

Upon resuming the connection, servers close the old connection.

### TLS

Clients may initially connect with TLS, only to later attempt to `RESUME` from a connection without TLS. In this situation, servers SHOULD deny the attempt with a message such as _"Cannot resume connection, client is no longer using TLS"_.

This is intended to ensure that TLS-only channel modes are not bypassed, and that clients prefer continuing to use TLS.


## Examples

A client with the nick `dan` reconnecting:

    C1 - C: PING 12345678
    C1 - S: :irc.example.com PONG 12345678
    C1 - C: PING 87654321
    ... C1 - Server does not send a response ...
    ... C1 disconnects and reconnects as C2 ...

    C2 - C: CAP LS
    C2 - C: NICK dan-
    C2 - C: USER d * 0 :An example user!
    C2 - S: :irc.example.com CAP * LS :sasl draft/resume
    C2 - C: CAP REQ :sasl draft/resume
    C2 - S: :irc.example.com CAP dan- ACK :sasl draft/resume
    C2 - C: RESUME dan 2017-04-13T15:12:51.620Z
    C2 - C: AUTHENTICATE PLAIN
    C2 - S: :irc.example.com AUTHENTICATE +
    C2 - C: AUTHENTICATE YnVubnkAYnVubnkAYnVubnk=
    C2 - S: :irc.example.com 900 dan- * bunny :You are now logged in as bunny
    C2 - S: :irc.example.com 903 dan- :SASL authentication successful
    C2 - S: :irc.example.com RESUMED dan ~d 127.0.0.1 2017-04-13T15:12:51.620Z
    ... C1's connection is closed and C1's attributes are applied to C2 ...
    C2 - S: :irc.example.com 001 dan :Welcome to the Internet Relay Network dan
    ... C2 receives regular registration burst ...
    C2 - S: :irc.example.com 376 dan :End of MOTD command
    C2 - S: :dan!~d@127.0.0.1 JOIN #test
    C2 - S: :irc.example.com 332 dan #test :Example topic
    C2 - S: :irc.example.com 333 dan #test george 1442060874
    C2 - S: :irc.example.com 353 dan @ #test :@dan @george +violet roger
    C2 - S: :irc.example.com MODE #test +o dan
