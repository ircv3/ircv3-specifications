---
title: `account-registration` Extension
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


## Motivation

The lack of a standardized way to register an account has been a source
of frustration for many participants for several years.

This isn't the first attempt to fix it. Unlike the withdrawn `ACC` proposal,
this effort aims to be more appealing to implementors by being severely
restricted in scope.
The only register-verify flows supported are those already in use in the wild.
The authors hope that this stripped-down solution will serve as the basis
for future organic expansion.

### Capability

This specification adds the `draft/account-registration` capability, whose
presence signifies that the server accepts the `REGISTER` command.

The capability has an optional value, a comma-separated list of key-value
pairs; the format is intended to follow the precedent set by
[the multiline draft][multiline].

The defined keys are:

 * `before-connect` - if present, indicates the server supports early
   registration, so it MUST NOT use
   `FAIL REGISTER COMPLETE_CONNECTION_REQUIRED`
 * `email-required` - if present, registrations require a valid email address
   to process
 * `custom-account-name` - if present, the account name can be different
   from the user's current nickname

Clients MUST ignore any value assigned to these keys, and MUST ignore
any unknown key.

Software implementing this work-in-progress specification MUST NOT use
the unprefixed `register` or `account-registration` capability names.
Instead, implementations SHOULD use the `draft/account-registration`
capability name to be interoperable with other software implementing
a compatible work-in-progress version.

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


# Client considerations

This section is non-normative.

Passwords are opaque byte strings.
It is recommended for them to be valid UTF-8;
or authentication may be impossible later (eg. with SASL PLAIN).
Servers may also choose to reject any non-UTF-8 password with
`UNACCEPTABLE_PASSWORD`.

As passwords may need to be sent in non-`authenticate` messages,
like a `PRIVMSG` to NickServ), which are limited in length, clients may want to
prevent or discourage users from setting passwords so long they may not fit
in these messages. 300 bytes should be a safe reasonable limit.


# Server considerations

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



[multiline]: https://github.com/ircv3/ircv3-specifications/pull/398/
