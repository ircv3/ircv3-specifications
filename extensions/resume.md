---
title: IRCv3 `resume` extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: Daniel Oakley
    period: 2017-2019
    email: daniel@danieloaks.net
  -
    name: Shivaram Lingamneni
    period: "2019"
    email: slingamn@cs.stanford.edu
  -
    name: "Jess"
    period: "2019"
    email: "jess@jesopo.uk"
---
## Introduction
Occasionally, clients disconnect from IRC. What normally happens is that the client connects with a different nickname, joins all their old channels, waits for the old connection to time out (or manually kills it using services), and then changes back to their original nickname.

The `resume` feature vastly simplifies this form of reconnection, for both the client that's reconnecting and clients joined to the same channels. In addition, this feature allows servers to either send missing chat history to the reconnecting client, or make other clients aware of just how much history has been lost.


## Architecture
This feature is enabled using the `draft/resume-0.4` capability, introduces the `RESUME` command, and uses the messages described below to convey state about the reconnection process and reconnecting clients. It also introduces the `BRB` command, which allows clients to close their connection while leaving their session on the server open for some time (to perform software upgrades, for example).

These commands use the [standard replies extension](https://github.com/ircv3/ircv3-specifications/pull/357) to relay warning information and indicate when they are not successful. The specific `FAIL` codes are given with each command's description.


### Capabilities
The `draft/resume-0.4` capability is advertised by servers that support resuming connections.

When this capability is negotiated, the server sends the client their resume token, which is used to authenticate future connection resumptions. Servers MAY extend the "ping timeout" length for that client (with the expectation that if the client disconnects, it will try to resume its connection as described here). Servers MAY also treat disconnections without `QUIT` as though the client used the `BRB` command described below.

Servers MUST ONLY generate and provide a resume token to a client when the client negotiates the `draft/resume-0.4` capability.

Capability negotiation example:

    C: CAP LS
    S: CAP * LS :multi-prefix draft/resume-0.4
    C: CAP REQ :draft/resume-0.4
    S: CAP * ACK :draft/resume-0.4
    S: RESUME TOKEN JKbAypzFiovffzuD8VEfcs6bOLrXsSenxsyZNt87Bb0YaQlyOzuQciz6cB2R


### Resuming Messages

#### `RESUME` Command
This command, sent from a client to a server, indicates that the client wishes to resume their old session. This command MAY ONLY be sent during connection registration.

    RESUME <token> [timestamp]

`<token>` is the old connection's resume token.

`<timestamp>`, if given, is a timestamp indicating when the client received the last message from the server on the old connection. This timestamp uses the same format as the IRCv3 `server-time` extension (i.e. `YYYY-MM-DDThh:mm:ss.sssZ`, or in UTC using extended format as specified by ISO 8601:2004(E) 4.3.2), and is passed to other clients to indicates how long the disconnection lasted.

If the request is successful, the server returns a `RESUME` message as described below, registration immediately completes, and the server begins replaying the current client state.

If the request is unsuccessful, the server returns a `FAIL RESUME` message with one of the codes below using the given format, including an appropriate description of the error:

| Code | Format |
| ---- | ------ |
| `INSECURE_SESSION` | `:<server> FAIL RESUME INSECURE_SESSION :Cannot resume connection, you are not connected with TLS` |
| `INVALID_TOKEN` | `:<server> FAIL RESUME INVALID_TOKEN :Cannot resume connection, token is not valid` |
| `REGISTRATION_IS_COMPLETED` | `:<server> FAIL RESUME REGISTRATION_IS_COMPLETED :Cannot resume connection, connection registration has completed` |
| `CANNOT_RESUME` | `:<server> FAIL RESUME CANNOT_RESUME :Cannot resume connection, for a different reason described here` |

If a client receives a `FAIL RESUME` message, regardless of the code, then they MUST abort the resume attempt and connect to the server normally instead.

#### `RESUME` Message
This message, sent from a server to a client, indicates that a `RESUME` request has been successful.

    RESUME <oldnick>

`<oldnick>` is the nickname of the session being resumed. After receiving this message, the client MUST assume that this is their nickname.

#### `RESUMED` Message
This message is sent by the server to indicate that another client has reconnected:

    :nick!user@oldhost RESUMED <host> [timestamp]

`<nick>` and `<oldhost>` indicate the client that has reconnected, and are the details of the old client. `<host>` indicates the reconnecting client's new hostname, and the receiving client MUST process this information as they would from a regular [`CHGHOST`](https://ircv3.net/specs/extensions/chghost-3.2.html) message. `<timestamp>`, if given, is a timestamp of the form described above which indicates the last message received by the reconnecting clent.

Upon receiving a `RESUMED` message, clients SHOULD display in some way that the given user has reconnected (as message history may have been lost and the users' chat may have been interrupted). If `<timestamp>` is given, clients SHOULD use this to display how much message history seems to have been lost.


### BRB Messages

#### `BRB` Command
This command, sent from a client to a server, indicates that the client wishes to disconnect from the server and resume their connection within some amount of time.

    BRB <reason>

`<reason>` is used as an `AWAY` reason for the client until either the client resumes their session or the window of time for resumption closes - in which case it will be used as the displayed `QUIT` reason.

If the request is successful, the server returns a `BRB` message as described below, and the client's connection to the server is immediately terminated.

If the request is unsuccessful, the server returns a `FAIL BRB` message with one of the codes below using the given format, including an appropriate description of the error:

| Code | Format |
| ---- | ------ |
| `CANNOT_BRB` | `:<server> FAIL BRB CANNOT_BRB :BRB is not supported` |

If a client receives a `FAIL BRB` message, regardless of the code, then they MUST assume the attempt has failed.

#### `BRB` Message
This message, sent from a server to a client, indicates that a `BRB` request has been successful.

    BRB <timeout>

`<timeout>` is an integer that represents how many seconds the client has to reconnect before their session is closed and they will no longer be able to resume. If the server cannot give an accurate duration, it should either approximate the length of time or give a zero `0` (representing that the server cannot estimate how long until the session is terminated).

After this message is returned, the client's connection to the server is immediately terminated.


## Capability Negotiation
The first step in successfully resuming a connection is for the 'old client' to negotiate the `draft/resume-0.4` capability and receive a resume token. This process is illustrated here:

    C: CAP LS
    S: CAP * LS :multi-prefix draft/resume-0.4
    C: CAP REQ :draft/resume-0.4
    S: CAP * ACK :draft/resume-0.4
    S: RESUME TOKEN JKbAypzFiovffzuD8VEfcs6bOLrXsSenxsyZNt87Bb0YaQlyOzuQciz6cB2R

Considerations around tokens and the process for generating them is described below in the [Resume Token](#resume-token) section.


## Resuming A Connection
When a client detects that it has become disconnected from a server, it SHOULD try to resume before it breaks the existing connection.

Upon establishing the new connection, the client begins capability negotiation, negotiates all mutually-supported capabilities, and MUST confirm that the `draft/resume-0.4` capability exists. If this capability does not exist, the client continues connection registration without attempting to resume. If this capability does exist, the client sends the `RESUME` command and MUST wait for either a `RESUME` or a `FAIL RESUME` message from the server before continuing registration. It should be noted that the client MUST NOT perform SASL authentication if the `draft/resume-0.4` capability exists and they wish to resume their session, as completing SASL auth will end connection registration and abort the resumption attempt.

If the token provided by the new client is validated by the server, the old client completed connection registration with the server, and both the old and new clients use TLS, then the attempt SHOULD be successful. If the attempt is successful, the server MUST send the client a `RESUME` message and complete connection registration immediately (at which time the state will begin to replay as described below). If the attempt is unsuccessful (for example, if either session is not using a secure connection), the server MUST send a `FAIL RESUME` message, and then allow the client to continue connection registration.

If the client receives a `RESUME SUCCESS` message, connection registration will complete immediately and state will begin to replay as described below. If the client receives a `FAIL RESUME`, the client MUST continue registration as though they are joining the server normally.

On a successful request, the server:

1. Terminates the old client's connection.
2. Updates the client's hostname and resume token (or lack of one) to match the new client's information.
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
- The old client's resume token (after completing the resume request, this token is thrown away), including any session-specific values that the server uses internally to create and validate the token.


## Resume Token
Upon indicating they support session resumption, clients are provided with a resume token. This token is used to authenticate future `RESUME` attempts with the server.

Clients MUST treat their resume token as an opaque value, and provide it to the server as given.

Servers use resume tokens to authenticate and allow a user to resume their old connection. The format of the token is implementation-defined, but MUST NOT contain any spaces and MUST NOT start with a colon character `(:)`, as these would break the protocol framing.

Described here is the 'typical method' of creating and validating resume tokens. Typical resume tokens contain two values:

- Resume ID: A random, unique value that identifies the client's session, which allows the server to look-up which session should be resumed.
- Resume Key: A random, cryptographically-secure value for the client's session, which is used to authenticate that the new client can resume the given existing session.

To validate a token during a resume attempt, a typical server implementation with a token as described above would:

1. Accept the resume token from the client.
2. Extract the ID and Key from the token.
3. Use the ID to look-up the session the client wishes to resume.
4. Use a **constant-time comparison function** to compare the provided key and the key of the session being resumed, to ensure they match.

This approach is recommended as it protects against timing attacks. Implementers MAY use their own methods of validating tokens, but are cautioned to think very carefully about security considerations and possible attacks against their method.


## Examples

### Successful Resumption
Successful `RESUME` attempt from a client with the nick `dan` reconnecting. The nickname the new client is connecting with is `dan-backup-nick`. The old connection used the username `~u` and the host `192.168.0.5`, and the new connection has the username `~d` and the host `10.0.0.3`:

    C1 - C: PING 12345678
    C1 - S: :irc.example.com PONG 12345678
    C1 - C: PING 87654321
    ... C1 - Server does not send a response ...
    ... C1 reconnects as C2 ...

    C2 - C: CAP LS
    C2 - C: NICK dan-backup-nick
    C2 - C: USER d * 0 :An example user!
    C2 - S: :irc.example.com CAP * LS :multi-prefix draft/resume-0.4 sasl
    C2 - C: CAP REQ :multi-prefix draft/resume-0.4 sasl
    C2 - S: :irc.example.com CAP dan-backup-nick ACK :multi-prefix draft/resume-0.4 sasl
    C2 - S: :irc.example.com RESUME TOKEN JKbAypzFiovffzuD8VEfcs6bOLrXsSenxsyZNt8
    C2 - C: RESUME A8KgnZPYDaRiGMzZWLu2frVvtN7lbCxO3hTwGLO 2017-04-13T15:12:51.620Z
    C2 - S: :irc.example.com RESUME dan
    ... C1's connection is closed and C1's attributes are applied to C2 ...
    C2 - S: :irc.example.com 001 dan :Welcome to the Internet Relay Network dan
    ... C2 receives regular registration burst ...
    C2 - S: :irc.example.com 376 dan :End of MOTD command
    C2 - S: :dan!~u@10.0.0.3 JOIN #test
    C2 - S: :irc.example.com 332 dan #test :Example topic
    C2 - S: :irc.example.com 333 dan #test george 1442060874
    C2 - S: :irc.example.com 353 dan @ #test :@dan @george +violet roger
    C2 - S: :irc.example.com MODE #test +o dan

Here is this successful reconnection seen by `george`, a client that does not have the `draft/resume-0.4` capability enabled:

    S: :dan!~u@192.168.0.5 QUIT :Client reconnected (24 seconds of message history lost)
    S: :dan!~u@10.0.0.3 JOIN #test
    S: :irc.example.com MODE #test +o dan

And here is this reconnection seen by `violet`, a client that has the `draft/resume-0.4` capability:

    S: :dan!~old@192.168.0.5 RESUMED 10.0.0.3 2017-04-13T15:12:51.620Z

### Failed Resumption
Failed `RESUME` attempt from a client with the nick `dan` reconnecting. The nickname the new client is connecting with is `dan-backup-nick`. The old connection used the username `~old` and the host `192.168.0.5`, and the new connection uses the username `~d` and the host `10.0.0.3`:

    C1 - C: PING 12345678
    C1 - S: :irc.example.com PONG 12345678
    C1 - C: PING 87654321
    ... C1 - Server does not send a response ...
    ... C1 reconnects as C2 ...

    C2 - C: CAP LS
    C2 - C: NICK dan-backup-nick
    C2 - C: USER d * 0 :An example user!
    C2 - S: :irc.example.com CAP * LS :multi-prefix draft/resume-0.4 sasl
    C2 - C: CAP REQ :multi-prefix draft/resume-0.4 sasl
    C2 - S: :irc.example.com CAP dan-backup-nick ACK :multi-prefix draft/resume-0.4 sasl
    C2 - S: :irc.example.com RESUME TOKEN JKbAypzFiovffzuD8VEfcs6bOLrXsSenxsyZNt8
    C2 - C: RESUME A8KgnZPYDaRiGMzZWLu2frVvtN7lbCxO3hTwGLO 2017-04-13T15:12:51.620Z
    C2 - S: :irc.example.com FAIL RESUME INVALID_TOKEN :Cannot resume connection, token is not valid
    C2 - C: AUTHENTICATE PLAIN
    C2 - S: :irc.example.com AUTHENTICATE +
    C2 - C: AUTHENTICATE YnVubnkAYnVubnkAYnVubnk=
    C2 - S: :irc.example.com 900 dan-backup-nick * bunny :You are now logged in as bunny
    C2 - S: :irc.example.com 903 dan-backup-nick :SASL authentication successful
    C2 - S: :irc.example.com 001 dan-backup-nick :Welcome to the Internet Relay Network dan-backup-nick
    ... regular connection ...

### Successful BRB
Successful `BRB` attempt from a client with the nickname `dan`:

    C: BRB :Software updates
    S: BRB 60
    ... client connection is immediately terminated ...

Other clients receive:

    S: :dan!user@host AWAY :Software updates

If `dan` does not resume their connection in time, their connection is closed and other clients receive something like this:

    S: :dan!user@host QUIT :Quit: Software updates

### Failed BRB
Failed `BRB` attempt from a client:

    C: BRB :Software updates
    S: :irc.example.com FAIL BRB CANNOT_BRB :BRB is unavailable at this time


## Implementation Considerations
This section notes considerations software authors will need to take into account while implementing this specification. This section is non-normative.

Server authors should allow clients to resume across server links on the same network. For example, if A and B are both servers on the same network, and a client who was on server A tries to resume when connected to server B, the server software should transfer the session across to server B silently.

When reconnecting and intending to use `RESUME`, clients should first try to reconnect to the same server / IP address before falling back to any server on the same network. Some server software may have difficulty transferring the connection between two different servers, so doing this can help give clients the best chance of successfully using this feature.

Right now, when clients detect that their connection to the server may have dropped they tend to send a `QUIT` command, close their current connection and then create a new connection to the server. In cases where the server supports resuming connections, clients may find it more useful to attempt to establish a new link to the server and resume the connection before closing their old one. If this is done, clients should be able to better take advantage of connection resumption.

In addition, users sometimes manually reconnect when they see that there is lag on their connection. In these cases, clients may also wish to do the above rather than closing the connection and then reconnecting.

When clients see a `RESUMED` message for another client which contains a timestamp, they can calculate how much time has passed since the timestamp and the current time and then display this next to the reconnect notice. Displaying this can assist users in knowing how much message history has been lost in private queries and channels.

A client that disconnects and reconnects to the server should explicitly display the reconnection, even if they're able to resume successfully. This is so that the user knows why they may be missing message history and similar issues.

Servers may wish to check the new hostmask of resuming clients, to ensure that it does not fall under their list of banned hosts or hostmasks.


## Security Considerations
This section notes security-specific considerations software authors will need to take into account while implementing this specification. This section is non-normative.

Servers should verify that clients cannot use `resume` to unintentionally bypass or evade any checks normally performed during registration, such as the `PASS` command or IP/nickmask bans. For this reason, we recommend only allowing clients to resume if their old session had completed connection registration successfully.

When servers apply the old client's session information to the new client, ensure that the new client retains their own unique resume token. Clients shouldn't share resume tokens under any circumstances.

Servers must construct resume tokens in a secure way. The [Resume Token](#resume-token) section above lays out a method that should prevent timing attacks. Any implementers creating their own method must protect against timing attacks and other possible attacks against their authentication method.

It is recommended that the token provide at least 128 bits of security strength. This reflects recommendations such as that of NIST [[1]](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-4/final) that modern systems target a 128-bit security level. In the relevant threat model of online attack by a classical adversary, this can be achieved through the use of a secret with 128 bits of entropy from a cryptographically secure pseudorandom source. With the 'typical method' of generating tokens, it is sufficient to ensure that the 'resume key' value of the token has this property.

Servers should decide whether, on resuming the session of an IRC operator, the old client's operator status should be applied to the new client. This may depend on other operator authentication considerations such as client certificates or similar.

Without this specification, if you know a client's account credentials you can typically close their connection (using something like `/NS GHOST` or `/NS REGAIN`).

With this specification, if you have a client's resume token you're able to see which hidden channels they've joined and essentially take-over their connection. Servers have to ensure that resume tokens are cryptographically-strong as they become the new baseline for authenticating as an active, online user. Clients should display incoming `RESUMED` messages in such a way that users are explicitly aware that the given client has reconnected.
 
