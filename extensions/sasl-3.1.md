---
title: IRCv3.1 SASL Authentication
layout: spec
updated-by:
  - sasl-3.2
copyrights:
  -
    name: "Jilles Tjoelker"
    period: "2009-2012"
    email: "jilles@stack.nl"
  -
    name: "William Pitcock"
    period: "2009-2012"
    email: "nenolod@dereferenced.org"
---

## Introduction

This document describes the client protocol for SASL authentication, as
implemented in Charybdis and Atheme.

The SASL protocol in general is documented in
[RFC 4422](https://tools.ietf.org/html/rfc4422), along with the 'EXTERNAL'
mechanism. The most commonly used 'PLAIN' mechanism is documented in
[RFC 4616](https://tools.ietf.org/html/rfc4616).

SASL authentication relies on the
[CAP client capability framework](../core/capability-negotiation.html).

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
later messages. Since there is no way besides ordering to make the difference
between these two forms, servers SHOULD avoid logging or formatting error
messages with the arguments of the AUTHENTICATE command to prevent secrets from
being leaked (e.g. in case a client doesn't wait for the server's initial empty
challenge before sending credentials).

The initial client message specifies the SASL mechanism to be used. (When this
is received, the IRCD will attempt to establish an association with a SASL
agent.) If this fails, a 904 numeric will be sent and the session state remains
unchanged; the client MAY try another mechanism. Otherwise, the server sends
a set of regular AUTHENTICATE messages with the initial server response.
If the chosen mechanism is client-first, the server sends an empty response
(`AUTHENTICATE +`, as described below).

    AUTHENTICATE <mechanism>

A set of regular AUTHENTICATE messages transmits a response from client to
server or vice versa. The server MAY intersperse other IRC protocol messages
between the AUTHENTICATE messages of a set.
The response is encoded in Base64
([RFC 4648](https://tools.ietf.org/html/rfc4648)), then split to 400-byte
chunks, and each chunk is sent as a separate `AUTHENTICATE` command. Empty
(zero-length) responses are sent as `AUTHENTICATE +`. If the last chunk was
exactly 400 bytes long, it must also be followed by `AUTHENTICATE +` to signal
end of response.
The server MAY place a limit on the total length of a response.

    *(AUTHENTICATE <400BASE64>)
    AUTHENTICATE <399BASE64 | '+'>

If the mechanism finishes with the server sending a non-empty challenge (such
as in SCRAM), clients MUST still send an empty response.

The client can abort an authentication by sending an asterisk as the data.
The server will send a 906 numeric.

    AUTHENTICATE '*'

If authentication fails, a 904 or 905 numeric will be sent and the
client MAY retry from the `AUTHENTICATE <mechanism>` command.
If authentication is successful, a 900 and 903 numeric will be sent.

If the client attempts to issue the AUTHENTICATE command after already
authenticating successfully, the server MUST reject it with a 907 numeric.

If the client completes registration (with CAP END, NICK, USER and any other
necessary messages) while the SASL authentication is still in progress, the
server SHOULD abort it and send a 906 numeric, then register the client
without authentication.

This document does not specify use of the AUTHENTICATE command in
registered (person) state.

## Example protocol exchange

C: indicates lines sent by the client, S: indicates lines sent by the server.

The client is using the PLAIN SASL mechanism with authentication identity
`jilles`, authorization identity `jilles` and password `sesame`.

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
    S: :jaguar.test 900 jilles jilles!jilles@localhost.stack.nl jilles :You are now logged in as jilles
    S: :jaguar.test 903 jilles :SASL authentication successful
    C: CAP END
    S: :jaguar.test 001 jilles :Welcome to the jillestest Internet Relay Chat Network jilles
    (usual welcome messages)

The client is using the SCRAM-SHA-1 mechanism.

    C: CAP REQ :sasl
    C: NICK jilles
    C: USER jilles cheetah.stack.nl 1 :Jilles Tjoelker
    S: NOTICE AUTH :*** Processing connection to jaguar.test
    S: NOTICE AUTH :*** Looking up your hostname...
    S: NOTICE AUTH :*** Checking Ident
    S: NOTICE AUTH :*** No Ident response
    S: NOTICE AUTH :*** Found your hostname
    S: :jaguar.test CAP jilles ACK :sasl
    C: AUTHENTICATE SCRAM-SHA-1
    S: AUTHENTICATE +
    C: AUTHENTICATE bixhPWppbGxlcyxuPWppbGxlcyxyPWM1UnFMQ1p5MEw0ZkdrS0FaMGh1akZCcw==
    S: AUTHENTICATE cj1jNVJxTENaeTBMNGZHa0tBWjBodWpGQnNYUW9LY2l2cUN3OWlEWlBTcGIscz01bUpPNmQ0cmpDbnNCVTFYLGk9NDA5Ng==
    C: AUTHENTICATE Yz1iaXhoUFdwcGJHeGxjeXc9LHI9YzVScUxDWnkwTDRmR2tLQVowaHVqRkJzWFFvS2NpdnFDdzlpRFpQU3BiLHA9T1ZVaGdQdTh3RW0yY0RvVkxmYUh6VlVZUFdVPQ==
    S: AUTHENTICATE dj1aV1IyM2M5TUppcjBaZ2ZHZjVqRXRMT242Tmc9
    C: AUTHENTICATE +
    S: :jaguar.test 900 jilles jilles!jilles@localhost.stack.nl jilles :You are now logged in as jilles
    S: :jaguar.test 903 jilles :SASL authentication successful
    C: CAP END
    S: :jaguar.test 001 jilles :Welcome to the jillestest Internet Relay Chat Network jilles
    (usual welcome messages)

Servers may also add a source to their AUTHENTICATE messages, just like any message.

    C: CAP REQ :sasl
    C: NICK jilles
    C: USER jilles cheetah.stack.nl 1 :Jilles Tjoelker
    S: NOTICE AUTH :*** Processing connection to jaguar.test
    S: NOTICE AUTH :*** Looking up your hostname...
    S: NOTICE AUTH :*** Checking Ident
    S: NOTICE AUTH :*** No Ident response
    S: NOTICE AUTH :*** Found your hostname
    S: :jaguar.test CAP jilles ACK :sasl
    C: AUTHENTICATE SCRAM-SHA-1
    S: :jaguar2.test AUTHENTICATE +
    C: AUTHENTICATE bixhPWppbGxlcyxuPWppbGxlcyxyPWM1UnFMQ1p5MEw0ZkdrS0FaMGh1akZCcw==
    S: :jaguar2.test AUTHENTICATE cj1jNVJxTENaeTBMNGZHa0tBWjBodWpGQnNYUW9LY2l2cUN3OWlEWlBTcGIscz01bUpPNmQ0cmpDbnNCVTFYLGk9NDA5Ng==
    C: AUTHENTICATE Yz1iaXhoUFdwcGJHeGxjeXc9LHI9YzVScUxDWnkwTDRmR2tLQVowaHVqRkJzWFFvS2NpdnFDdzlpRFpQU3BiLHA9T1ZVaGdQdTh3RW0yY0RvVkxmYUh6VlVZUFdVPQ==
    S: :jaguar2.test AUTHENTICATE dj1aV1IyM2M5TUppcjBaZ2ZHZjVqRXRMT242Tmc9
    C: AUTHENTICATE +
    S: :jaguar.test 900 jilles jilles!jilles@localhost.stack.nl jilles :You are now logged in as jilles
    S: :jaguar.test 903 jilles :SASL authentication successful
    C: CAP END
    S: :jaguar.test 001 jilles :Welcome to the jillestest Internet Relay Chat Network jilles
    (usual welcome messages)

Alternatively the client could request the list of capabilities and enable
an additional capability.

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
    S: :jaguar.test 900 jilles jilles!jilles@localhost.stack.nl jilles :You are now logged in as jilles
    S: :jaguar.test 903 jilles :SASL authentication successful
    C: CAP END
    S: :jaguar.test 001 jilles :Welcome to the jillestest Internet Relay Chat Network jilles
    (usual welcome messages)

## Numerics used by this extension

`900` aka `RPL_LOGGEDIN` is sent when the user's account name is set (whether by SASL or otherwise).

    :server 900 <nick> <nick>!<ident>@<host> <account> :You are now logged in as <user>

`901` aka `RPL_LOGGEDOUT` is sent when the user's account name is unset (whether by SASL or otherwise).

    :server 901 <nick> <nick>!<ident>@<host> :You are now logged out

`902` aka `ERR_NICKLOCKED` is sent when the SASL authentication fails because the account is currently locked out, held, or otherwise administratively made unavailable.

    :server 902 <nick> :You must use a nick assigned to you

`903` aka `RPL_SASLSUCCESS` is sent when the SASL authentication finishes successfully. It usually goes along with `900`.

    :server 903 <nick> :SASL authentication successful

`904` aka `ERR_SASLFAIL` is sent when the SASL authentication fails because of invalid credentials or other errors not explicitly mentioned by other numerics.

    :server 904 <nick> :SASL authentication failed

`905` aka `ERR_SASLTOOLONG` is sent when credentials are valid, but the SASL authentication fails because the client-sent `AUTHENTICATE` command was too long (i.e. the parameter longer than 400 bytes).

    :server 905 <nick> :SASL message too long

`906` aka `ERR_SASLABORTED` is sent when the SASL authentication is aborted because the client sent an `AUTHENTICATE` command with `*` as the parameter.

    :server 906 <nick> :SASL authentication aborted

`907` aka `ERR_SASLALREADY` is sent when the client attempts to initiate SASL authentication after it has already finished successfully for that connection.

    :server 907 <nick> :You have already authenticated using SASL

`908` aka `RPL_SASLMECHS` MAY be sent in reply to an `AUTHENTICATE` command which requests an unsupported mechanism. The numeric contains a comma-separated list of mechanisms supported by the server (or network, services).

    :server 908 <nick> <mechanisms> :are available SASL mechanisms

_(The final "text" parameter is not to be machine-parsed, as it tends to vary
between implementations and translations.)_

# Errata

* Previous versions of this specification mentioned that 904 numeric would be
  used when SASL was aborted. 906 ERR_SASLABORTED should be used when SASL is
  aborted.
* Previous versions of this specification did not precisely describe when
is RPL_SASLMECHS being sent.
* Clarified the language how responses are transmitted.
* Added empty initial server response for client-first mechanisms. This had happened de-facto already.
* Added empty final client response for certain mechanisms. This had happened de-facto already.
* Previous versions of this specification precisely specified the serialization,
  de-jure making it stricter than regular IRC command (disallowing colons).
  It now uses a higher-level grammar similar to other IRCv3 specifications,
  which assumes messages are parsed using RFC1459-like deserialisation,
  as this is how it is was already implemented and understood in practice.
