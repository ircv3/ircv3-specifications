# SASL authentication

This document describes the client protocol for SASL authentication, as
implemented in charybdis and atheme.

SASL authentication relies on the
[CAP client capability framework](/specification/capability-negotiation-3.1).

Support for SASL authentication is indicated with the "sasl" capability.
The client MUST enable the sasl capability before using the AUTHENTICATE
command defined by this specification.

## The AUTHENTICATE command

The AUTHENTICATE command MUST be used before registration is complete and
with the sasl capability enabled. To enforce the former, it is RECOMMENDED
to only send CAP END when the SASL exchange is completed or needs to be
aborted. Clients SHOULD be prepared for timeouts at all times during the SASL
authentication.

There are two forms of the AUTHENTICATE command: initial client message and
later messages.

The initial client message specifies the SASL mechanism to be used. (When this
is received, the IRCD will attempt to establish an association with a SASL
agent.) If this fails, a 904 numeric will be sent and the session state remains
unchanged; the client MAY try another mechanism. Otherwise, the server sends
a set of regular AUTHENTICATE messages with the initial server response.

`initial-authenticate = "AUTHENTICATE" SP mechanism CRLF`

A set of regular AUTHENTICATE messages transmits a response from client to
server or vice versa. The server MAY intersperse other IRC protocol messages
between the AUTHENTICATE messages of a set. The "+" form is used for an empty
response. The server MAY place a limit on the total length of a response.

```regular-authenticate-set = *("AUTHENTICATE" SP 400BASE64 CRLF)
	"AUTHENTICATE" SP (1*399BASE64 / "+") CRLF```

The client can abort an authentication by sending an asterisk as the data.
The server will send a 904 numeric.

`authenticate-abort = "AUTHENTICATE" SP "*" CRLF`

If authentication fails, a 904 or 905 numeric will be sent and the
client MAY retry from the AUTHENTICATE <mechanism> command.
If authentication is successful, a 900 and 903 numeric will be sent.

If the client attempts to issue the AUTHENTICATE command after already
authenticating successfully, the server MUST reject it with a 907 numeric.

If the client completes registration (with CAP END, NICK, USER and any other
necessary messages) while the SASL authentication is still in progress, the
server SHOULD abort it and send a 906 numeric, then register the client
without authentication.

This document does not specify use of the AUTHENTICATE command in
registered (person) state.

Example protocol exchange

C: indicates lines sent by the client, S: indicates lines sent by the server.

The client is using the PLAIN SASL mechanism with authentication identity
jilles, authorization identity jilles and password sesame.

<pre>
C: CAP REQ :sasl
C: NICK jilles
C: USER jilles cheetah.stack.nl 1 :Jilles Tjoelker
S: NOTICE AUTH :*** Processing connection to jaguar.test
S: NOTICE AUTH :*** Looking up your hostname...
S: NOTICE AUTH :*** Checking Ident
S: NOTICE AUTH :*** No Ident response
S: NOTICE AUTH :*** Found your hostname
S: :jaguar.test CAP jilles ACK :sasl 
C: AUTHENTICATE PLAIN
S: AUTHENTICATE +
C: AUTHENTICATE amlsbGVzAGppbGxlcwBzZXNhbWU=
S: :jaguar.test 900 jilles jilles!jilles@localhost.stack.nl jilles :You are now logged in as jilles.
S: :jaguar.test 903 jilles :SASL authentication successful
C: CAP END
S: :jaguar.test 001 jilles :Welcome to the jillestest Internet Relay Chat Network jilles
(usual welcome messages)
</pre>

Alternatively the client could request the list of capabilities and enable
an additional capability.

<pre>
C: CAP LS
C: NICK jilles
C: USER jilles cheetah.stack.nl 1 :Jilles Tjoelker
S: NOTICE AUTH :*** Processing connection to jaguar.test
S: NOTICE AUTH :*** Looking up your hostname...
S: NOTICE AUTH :*** Checking Ident
S: NOTICE AUTH :*** No Ident response
S: NOTICE AUTH :*** Found your hostname
S: :jaguar.test CAP * LS :multi-prefix sasl
C: CAP REQ :multi-prefix sasl
S: :jaguar.test CAP jilles ACK :multi-prefix sasl 
C: AUTHENTICATE PLAIN
S: AUTHENTICATE +
C: AUTHENTICATE amlsbGVzAGppbGxlcwBzZXNhbWU=
S: :jaguar.test 900 jilles jilles!jilles@localhost.stack.nl jilles :You are now logged in as jilles.
S: :jaguar.test 903 jilles :SASL authentication successful
C: CAP END
S: :jaguar.test 001 jilles :Welcome to the jillestest Internet Relay Chat Network jilles
(usual welcome messages)
</pre>

## Numerics used by this extension

`900` aka RPL\_LOGGEDIN is sent when the user's account name is set (whether by SASL or otherwise).

    :server 900 <nick> <nick>!<ident>@<host> <account> :You are now logged in as <user>

`901` aka RPL\_LOGGEDOUT is sent when the user's account name is unset (whether by SASL or otherwise).

    :server 901 <nick> <nick>!<ident>@<host> :You are now logged out

`902` aka ERR\_NICKLOCKED is sent when the SASL authentication fails because the account is currently locked out, held, or otherwise administratively made unavailable.

    :server 902 <nick> :You must use a nick assigned to you.

`903` aka RPL\_SASLSUCCESS is sent when the SASL authentication finishes successfully. It usually goes along with `900`.

    :server 903 <nick> :SASL authentication successful

`904` aka ERR\_SASLFAIL is sent when the SASL authentication fails because of invalid credentials or other errors not explicitly mentioned by other numerics.

    :server 904 <nick> :SASL authentication failed

`905` aka ERR\_SASLTOOLONG is sent when credentials are valid, but the SASL authentication fails because the client-sent `AUTHENTICATE` command was too long (i.e. the parameter longer than 400 bytes).

    :server 905 <nick> :SASL message too long

`906` aka ERR\_SASLABORTED is sent when the SASL authentication is aborted because the client sent an `AUTHENTICATE` command with `*` as the parameter.

    :server 906 <nick> :SASL authentication aborted

`907` aka ERR\_SASLALREADY is sent when the client attempts to initiate SASL authentication after it has already finished successfully for that connection.

    :server 907 <nick> :You have already authenticated using SASL

`908` aka RPL\_SASLMECHS is sent when the client requests a list of SASL mechanisms supported by the server (or network, services). The numeric contains a comma-separated list of mechanisms.

    :server 908 <nick> <mechanisms> :are available SASL mechanisms
