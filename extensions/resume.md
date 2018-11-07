---
title: IRCv3 `resume` extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: Daniel Oakley
    period: 2017-2018
    email: daniel@danieloaks.net
---
## Introduction

Occasionally, clients disconnect from IRC. What normally happens is that the client connects with a different nickname, joins all their old channels, waits for the old connection to time out (or manually kills it using services), and then changes back to their original nickname.

The `resume` feature vastly simplifies this form of reconnection, for both the client that's reconnecting and clients joined to the same channels. In addition, this feature allows servers to either send missing chat history to the reconnecting client, or make other clients aware of just how much history has been lost.


## Architecture

This feature is enabled using the `draft/resume-0.2` capability, introduces the `RESUME` command, and uses the messages described below to convey state about the reconnection process and reconnecting clients.

### Resumption Token

Upon indicating they support session resumption, clients are provided with a resumption token. This token is used to authenticate future `RESUME` attempts.

Servers MUST ensure that this token is long, cryptographically secure, and not able to be easily brute-forced as this token will protect the security of clients' information. Resumption tokens are case-sensitive, and clients MUST treat them as opaque values.

The token MUST NOT contain any spaces and MUST NOT start with a colon character `(:)`, as these would break the protocol framing. There is no sort of encoding or decoding defined for the token.

### Capabilities

The `draft/resume-0.2` capability is advertised by servers that support resuming connections.

When this capability is negotiated, the server sends the client the resumption token, which is used to authenticate future connection resumptions. In addition, servers MAY extend the "ping timeout" length for that client (with the expectation that if the client disconnects, it will try to resume its connection as described here).

Servers MAY ONLY generate and provide a resumption token to a client when the client negotiates the `draft/resume-0.2` capability.

Capability negotiation example:

    C: CAP LS
    S: CAP * LS :multi-prefix draft/resume-0.2
    C: CAP REQ :draft/resume-0.2
    S: CAP * ACK :draft/resume-0.2
    S: RESUME TOKEN JKbAypzFiovffzuD8VEfcs6bOLrXsSenxsyZNt87Bb0YaQlyOzuQciz6cB2R

### Commands

#### `RESUME` Command

This command MAY ONLY be sent by a client during connection registration, and indicates that the client wishes to resume their session.

    RESUME <oldnick> <token> [timestamp]

`<oldnick>` is their old nickname, and `<token>` is the old connection's resumption token.

`<timestamp>`, if given, is a timestamp indicating when the client received the last message from the server on the old connection. This timestamp uses the same format as the IRCv3 `server-time` extension (i.e. `YYYY-MM-DDThh:mm:ss.sssZ`, or in UTC using extended format as specified by ISO 8601:2004(E) 4.3.2), and is passed to other clients to indicates how long the disconnection lasted.

### Messages

The `RESUME` message is used to indicate information and the result of a `RESUME` attempt back to the client who attempted it. This message has various sub-messages, indicated below.

If a client does not recognise a `RESUME` subcommand they receive, they `MUST` silently ignore the subcommand.

#### `RESUME SUCCESS` Message

    RESUME SUCCESS <oldnick>

The `SUCCESS` message indicates that the attempt to resume the connection from `<oldnick>` was successful.

#### `RESUME WARN` Message

    RESUME WARN <oldnick> <info>

The `WARN` message indicates that there was a warning about the resume attempt, but not that it has succeeded or failed (it will be followed up by a `SUCCESS` or `ERR`). `<info>` is an information text string indicating the reson for the warning (e.g.: `"Chat history may be truncated"`).

#### `RESUME ERR` Message

    RESUME ERR <oldnick> <info>

The `ERR` message indicates that the attempt to resume the connection from `<oldnick>` failed. `<info>` is an information text string indicating the reson for the failure (e.g.: `"Cannot resume connection, token is not valid"`).

#### `RESUMED` Message

    :nick!olduser@oldhost RESUMED <user> <host> [timestamp]

This message is sent to indicate that a client has reconnected. `<nick>`, `<olduser>`, and `<oldhost>` indicate the client that has reconnected, and are the details of the old client. `<user>` and `<host>` indicate the reconnecting client's new username and hostname, and the client MUST process this information as they would a regular [`CHGHOST`](https://ircv3.net/specs/extensions/chghost-3.2.html) message. `<timestamp>`, if given, is a timestamp of the form described above which indicates the last message received by the reconnecting clent.

Upon receiving a `RESUMED` message, clients MUST display in some way that the given user has reconnected. If `<timestamp>` is given, clients SHOULD use this to display how much message history seems to have been lost.


## Capability Negotiation

The first step in successfully resuming a connection is for the 'old client' to negotiate the `draft/resume-0.2` capability and receive a resumption token. This process is illustrated here:

    C: CAP LS
    S: CAP * LS :multi-prefix draft/resume-0.2
    C: CAP REQ :draft/resume-0.2
    S: CAP * ACK :draft/resume-0.2
    S: RESUME TOKEN JKbAypzFiovffzuD8VEfcs6bOLrXsSenxsyZNt87Bb0YaQlyOzuQciz6cB2R


## Resuming A Connection

When a client detects that it has become disconnected from a server, it SHOULD try to resume before it breaks the existing connection.

Upon establishing the new connection, the client begins capability negotiation, negotiates supported capabilities, and MUST confirm that the `draft/resume-0.2` capability exists. If this capability does not exist, the client continues connection registration without attempting to resume. If this capability does exist, the client sends the `RESUME` command and MUST wait for either a `RESUME SUCCESS` or a `RESUME ERR` message before continuing registration. It should be noted that the client MUST NOT perform SASL authentication if the `draft/resume-0.2` capability exists, as this will end connection registration and abort the resumption attempt.

If the token provided by the new client matches the old client's resumption token, and both the old and new clients use TLS, then the attempt SHOULD be successful. If the attempt is successful, the server MUST send a `RESUME SUCCESS` message and complete connection registration immediately. If the attempt is unsuccessful, the server MUST send a `RESUME ERR` message and allow the server to continue connection registration.

If the client receives a `RESUME SUCCESS` message, connection registration will complete immediately and state will begin to replay. If the client receives a `RESUME ERR`, the client MUST continue registration as though they are joining the server normally.

On a successful request, the server:

1. Terminates the old client's connection.
2. Updates the client's username, hostname, and resume token (or lack of one) to match the new client's information.
3. Plays the 'joining server' burst to the new client.
4. Replays the client's session to the new client.
5. Sends `RESUMED` or `QUIT`+`JOIN` messages to other clients as appropriate.
6. Sends clients that have the reconnecting user `MONITOR`'d one `RPL_MONOFFLINE` numeric and one `RPL_MONONLINE` numeric indicating that the user has reconnected. The timestamps on both these messages, if sent, SHOULD be the current time.
7. If message history could not be replayed (because it is not stored, or for any other reason), sends the new client a `RESUME WARN` message making them aware of this.

On a successful request, the session information that MUST be applied from the old client and replayed includes, but is not limited to:

- Joined channels, along with channel membership prefixes (op, halfop, etc).
- Metadata.
- MONITOR list.
- User modes.
- Logged-in user account.

Session information that MUST NOT be applied from the old client and replayed includes:

- Enabled client capabilities.
- The old client's resume token (after completing the resume request, this token is thrown away).

If the resume request is unsuccessful, the server continues as though the new client is a normal client joining the server.


## Examples

### Successful Resumption

Successful attempt from a client with the nick `dan` reconnecting. The old connection used the username `~old` and the host `192.168.0.5`, and the new connection uses the username `~d` and the host `10.0.0.3`:

    C1 - C: PING 12345678
    C1 - S: :irc.example.com PONG 12345678
    C1 - C: PING 87654321
    ... C1 - Server does not send a response ...
    ... C1 reconnects as C2 ...

    C2 - C: CAP LS
    C2 - C: NICK dan-backup-nick
    C2 - C: USER d * 0 :An example user!
    C2 - S: :irc.example.com CAP * LS :multi-prefix draft/resume-0.2 sasl
    C2 - C: CAP REQ :multi-prefix draft/resume-0.2 sasl
    C2 - S: :irc.example.com CAP dan-backup-nick ACK :multi-prefix draft/resume-0.2 sasl
    C2 - S: :irc.example.com RESUME TOKEN JKbAypzFiovffzuD8VEfcs6bOLrXsSenxsyZNt8
    C2 - C: RESUME dan A8KgnZPYDaRiGMzZWLu2frVvtN7lbCxO3hTwGLO 2017-04-13T15:12:51.620Z
    C2 - S: :irc.example.com RESUME SUCCESS dan
    ... C1's connection is closed and C1's attributes are applied to C2 ...
    C2 - S: :irc.example.com 001 dan :Welcome to the Internet Relay Network dan
    ... C2 receives regular registration burst ...
    C2 - S: :irc.example.com 376 dan :End of MOTD command
    C2 - S: :dan!~d@127.0.0.1 JOIN #test
    C2 - S: :irc.example.com 332 dan #test :Example topic
    C2 - S: :irc.example.com 333 dan #test george 1442060874
    C2 - S: :irc.example.com 353 dan @ #test :@dan @george +violet roger
    C2 - S: :irc.example.com MODE #test +o dan

Here is this successful reconnection seen by `george`, a client that does not have the `draft/resume-0.2` capability enabled:

    S: :dan!~old@192.168.0.5 QUIT :Client reconnected (24 seconds of message history lost)
    S: :dan!~d@127.0.0.1 JOIN #test
    S: :irc.example.com MODE #test +o dan

And here is this reconnection seen by `violet`, a client that has the `draft/resume-0.2` capability:

    S: :dan!~old@192.168.0.5 RESUMED dan ~d 10.0.0.3 2017-04-13T15:12:51.620Z

### Failed Resumption

Failed attempt from a client with the nick `dan` reconnecting. The old connection used the username `~old` and the host `192.168.0.5`, and the new connection uses the username `~d` and the host `10.0.0.3`:

    C1 - C: PING 12345678
    C1 - S: :irc.example.com PONG 12345678
    C1 - C: PING 87654321
    ... C1 - Server does not send a response ...
    ... C1 reconnects as C2 ...

    C2 - C: CAP LS
    C2 - C: NICK dan-backup-nick
    C2 - C: USER d * 0 :An example user!
    C2 - S: :irc.example.com CAP * LS :multi-prefix draft/resume-0.2 sasl
    C2 - C: CAP REQ :multi-prefix draft/resume-0.2 sasl
    C2 - S: :irc.example.com CAP dan-backup-nick ACK :multi-prefix draft/resume-0.2 sasl
    C2 - S: :irc.example.com RESUME TOKEN JKbAypzFiovffzuD8VEfcs6bOLrXsSenxsyZNt8
    C2 - C: RESUME dan A8KgnZPYDaRiGMzZWLu2frVvtN7lbCxO3hTwGLO 2017-04-13T15:12:51.620Z
    C2 - S: :irc.example.com RESUME ERR :Cannot resume connection, token is not valid
    C2 - C: AUTHENTICATE PLAIN
    C2 - S: :irc.example.com AUTHENTICATE +
    C2 - C: AUTHENTICATE YnVubnkAYnVubnkAYnVubnk=
    C2 - S: :irc.example.com 900 dan-backup-nick * bunny :You are now logged in as bunny
    C2 - S: :irc.example.com 903 dan-backup-nick :SASL authentication successful
    C2 - S: :irc.example.com 001 dan-backup-nick :Welcome to the Internet Relay Network dan-backup-nick
    ... regular connection ...


## Implementation Considerations

This section notes considerations software authors will need to take into account while implementing this specification. This section is non-normative.

Right now, when clients detect that their connection to the server may have dropped they tend to send a `QUIT` command, close their current connection and then create a new connection to the server. In cases where the server supports resuming connections, clients may find it more useful to attempt to establish a new link to the server and resume the connection before closing their old one. If this is done, clients should be able to better take advantage of connection resumption.

In addition, users sometimes manually reconnect when they see that there is lag on their connection. In these cases, clients may also wish to do the above rather than closing the connection and then reconnecting.

When clients see a `RESUMED` message for another client which contains a timestamp, they can calculate how much time has passed since the timestamp and the current time and then display this next to the reconnect notice. Displaying this can assist users in knowing how much message history has been lost in private queries and channels.

A client that disconnects and reconnects to the server should explicitly display the reconnection, even if they're able to resume successfully. This is so that the user knows why they may be missing message history and similar issues.

Servers may wish to check the new hostmask of resuming clients, to ensure that it does not fall under their list of banned hosts or hostmasks.


## Security Considerations

This section notes security-specific considerations software authors will need to take into account while implementing this specification. This section is non-normative.

When servers apply the old client's session information to the new client, ensure that the new client retains their own unique resumption token. Clients shouldn't share resumption tokens under any circumstances.

Servers should decide whether, on resuming the session of an IRC operator, the old client's operator status should be applied to the new client. This may depend on other operator authentication considerations such as client certificates or similar.

Without this specification, if you know a client's account credentials you can typically close their connection (using something like `/NS GHOST` or `/NS REGAIN`).

With this specification, if you have a client's resume token you're able to see which hidden channels they've joined and essentially take-over their connection. Servers have to ensure that resume tokens are cryptographically-strong as they become the new baseline for authenticating as an active, online user. Clients should display incoming `RESUMED` messages in such a way that users are explicitly aware that the given client has reconnected.
 