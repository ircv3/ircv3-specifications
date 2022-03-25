---
title: "`account-registration` Extension"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Ed Kellett"
    period: "2020"
    email: "e@kellett.im"
  -
    name: "Val Lorentz"
    period: "2021"
    email: "progval+ircv3@progval.net"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use
the unprefixed `account-registration` capability name.
Instead, implementations SHOULD use the `draft/account-registration`
capability name to be interoperable with other software implementing
a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Motivation

A standard method of registering a user account has historically been missing
from IRC. Instead various methods such as sending messages to services bots
(e.g. NickServ) have been employed.

This specification aims to provide a standard baseline for account
registration that allows client developers to build more reliable and
integrated user experiences around account registration.

It does not aim to cover all registration-related use cases that may currently
be in use by server implementations to varying degrees. However, future
extensions to the specification might be proposed given a broad enough
consensus.

## Architecture

### Dependencies

This specification uses [standard replies][] framework.

### Capabilities

This specification adds the `draft/account-registration` capability, whose
presence signifies that the server accepts the `REGISTER` command.

The capability MAY be advertised by servers with an OPTIONAL value:
a comma (`,`) (0x2C) separated list of tokens. Each token consists of a key
with an optional value attached. If there is a value attached, the value
is separated from the key by an equals sign (`=`) (0x3D).
That is, `<key>[=<value>][,<key2>[=<value2>][,<keyN>[=<valueN>]]]`.
Keys specified in this document MUST only occur at most once.

Clients MUST ignore every token with a key that they don't understand,
and MUST ignore any value in these tokens.

The only defined capability keys so far are:

 * `before-connect` - if present, indicates the server supports early
   registration, so it MUST NOT use
   `FAIL REGISTER COMPLETE_CONNECTION_REQUIRED`
 * `email-required` - if present, registrations require a valid email address
   to process
 * `custom-account-name` - if present, the account name can be different
   from the user's current nickname

This specification also defines the informational `draft/account-required`
capability. If present, it indicates that the connection to the server cannot
be completed unless the clients authenticates, typically via SASL. Note, the
absence of this capability does not indicate that connection registration can
be completed without authentication; it may be disallowed due to specific
properties of the connection (e.g. an untrustworthy IP address), which will be
indicated instead by `FAIL * ACCOUNT_REQUIRED`. If the capability value
`draft/account-registration=before-connect` is advertised, clients should
respond to both of these conditions by suggesting that the user register an
account. Clients MUST NOT request `draft/account-required`, servers MUST reject
any `CAP REQ` command including this capability.

### Commands

    REGISTER <account> {<email> | "*"} <password>
    
The `REGISTER` command informs the server of a request to register
an account named for the current nick of the requestor.

If `<account>` is `*`, then this value is the user's current nickname.
If the server advertises the `custom-account-name` key, then this desired
account name can be different from the user's current nickname.

The `REGISTER` command MAY be sent at any point during the connection
that the client has a valid nickname.
If the client sends `REGISTER` before completing connection registration,
and receives a `FAIL REGISTER COMPLETE_CONNECTION_REQUIRED`, it SHOULD make
a second attempt after it receives the welcome message.

    VERIFY <account> <code>
    
The `VERIFY` command completes a registration that required verification
(eg. via email or CAPTCHA).

### Responses

    REGISTER SUCCESS <account> <message>
    VERIFY SUCCESS <account> <message>
    
Sent by the server when the `REGISTER` or `VERIFY` fully succeeded, and no
further action is needed.
Afterward, the client is assumed to be authenticated to the `<account>`
as if it used SASL.

    REGISTER VERIFICATION_REQUIRED <account> <message>
    
Sent by the server as a response to `REGISTER` when further action is required
to complete registration of the `<account>`, as explain in the human-readable
`<message>`.

    FAIL REGISTER ACCOUNT_EXISTS <account> <message>

Sent by the server as a response to `REGISTER` when the client tried to register
while using a nick that is not available as a new account's name
because someone already registered it.

The server MUST NOT send this response before the client sent a `NICK` command.

The client MAY try registering again later.
Clients should not automatically retry immediately without changing
their nickname or waiting.

    FAIL REGISTER BAD_ACCOUNT_NAME <account> <message>

Sent by the server to indicate that the desired account name is invalid or
otherwise restricted/disallowed. The message should tell the user how or why
the desired name has been rejected.

    FAIL REGISTER ACCOUNT_NAME_MUST_BE_NICK <account> <message>

Sent by the server to indicate that the desired account name does not match
the user's current nick, when it must match.

    FAIL REGISTER NEED_NICK * <message>

Sent by the server if the client sends `REGISTER` before sending a NICK command.
This is only possible in setups that advertise the `before-connect` key.

    FAIL REGISTER ALREADY_AUTHENTICATED <account> <message>
    FAIL VERIFY ALREADY_AUTHENTICATED <account> <message>

Sent by the server if the client is already authenticated.

    FAIL REGISTER WEAK_PASSWORD <account> <message>

Sent by the server if the password is considered too weak.

The client MAY try registering again.

    FAIL REGISTER UNACCEPTABLE_PASSWORD <account> <message>

Sent by the server if the password is invalid for any reason other than
weakness.

The client MAY try registering again.

    FAIL REGISTER INVALID_EMAIL <account> <message>

Sent by the server if it cannot send emails to the provided address.

    FAIL REGISTER UNACCEPTABLE_EMAIL <account> <message>

Sent by the server if the email address is valid, but not available for
account registration.

    FAIL REGISTER COMPLETE_CONNECTION_REQUIRED <message>
    FAIL VERIFY COMPLETE_CONNECTION_REQUIRED <message>

Sent by the server if the client sent a `REGISTER` command before connection
registration.
The server MUST NOT sent these replies if it advertizes the `before-connect`
key of the `draft/account-registration` capability.

    FAIL VERIFY INVALID_CODE <account> <message>

Sent by the server if the `<code>` in the `VERIFY` command is invalid
or expired.

    FAIL REGISTER TEMPORARILY_UNAVAILABLE <account> <message>
    FAIL VERIFY TEMPORARILY_UNAVAILABLE <account> <message>

Sent by the server if the `REGISTER`/`VERIFY` commands are temporarily
unavailable.

    FAIL * ACCOUNT_REQUIRED <message>

Sent by the server in response to the end of connection registration (e.g.
to `USER`, `NICK`, or `CAP END`) if connection registration cannot proceed
unless the user logs into an account with SASL.

## Examples

### While connected

A client with nick `tester` requests registration of an account named `test`:

    C: CAP LS 302
    S: CAP * LS :draft/account-registration=before-connect,email-required
    C: CAP REQ :draft/account-registration
    C: CAP END
    ...
    C: REGISTER test tester@example.org hunter2
    S: REGISTER VERIFICATION_REQUIRED test :Account created, pending verification; verification code has been sent to tester@example.org

The client then inputs the code sent by the server:

    C: VERIFY test 39gvcdg4myvnmdcfhvd6exsv4n
    S: VERIFY SUCCESS test :Account successfully registered

### `before-connect`

A client connects and asks to register an account named after its current nick:

    C: CAP LS 302
    C: NICK tester
    C: USER tester * * :Tester
    S: CAP * LS :draft/account-registration=before-connect,email-required
    C: CAP REQ :draft/account-registration
    S: CAP * ACK draft/account-registration
    C: REGISTER * tester@example.org hunter2
    S: REGISTER VERIFICATION_REQUIRED tester :Account created, pending verification; verification code has been sent to tester@example.org

The client then inputs the code sent by the server, and the server
immediately authenticates it:

    C: VERIFY tester 39gvcdg4myvnmdcfhvd6exsv4n
    S: VERIFY SUCCESS tester :Account successfully registered
    S: 900 * * tester :You are now logged in as tester

The client can then proceed with the connection:

    C: CAP END
    S: 001 tester :Welcome to the IRC network
    ...

### With no email or verification

    C: CAP LS 302
    S: CAP * LS :draft/account-registration=before-connect
    C: CAP REQ :draft/account-registration
    C: CAP END
    ...
    C: REGISTER * * hunter2
    S: REGISTER SUCCESS tester :Account successfully registered

### With `email-required`, but not verified

    C: CAP LS 302
    S: CAP * LS :draft/account-registration=before-connect,email-required
    C: CAP REQ :draft/account-registration
    C: CAP END
    ...
    C: REGISTER * tester@example.org hunter2
    S: REGISTER SUCCESS tester :Account successfully registered

## Client considerations

This section is non-normative.

Passwords are opaque byte strings. It is recommended for them to be valid UTF-8;
or authentication may be impossible later (eg. with SASL PLAIN).
Servers may also choose to reject any non-UTF-8 password with `UNACCEPTABLE_PASSWORD`.

As passwords may need to be sent in non-`authenticate` messages,
like a `PRIVMSG` to NickServ), which are limited in length, clients may want to
prevent or discourage users from setting passwords so long they may not fit
in these messages. 300 bytes should be a safe reasonable limit.


## Server considerations

This section is non-normative.

Server implementations should be careful not to accidentally send `INVALID_EMAIL`
as a response to a valid address.

As passwords may need to be sent in non-`authenticate` messages,
like a `PRIVMSG` to NickServ), which are limited in length, servers may want to
prevent or discourage users from setting passwords so long they may not fit
in these messages. 300 bytes should be a safe reasonable limit.

Servers should ensure that if they allow REGISTER before connection registration,
this functionality cannot be used to bypass any authentication restrictions
defined in the server configuration, e.g. requirements that clients send
a server password with PASS.

[standard replies]: ../extensions/standard-replies.html
