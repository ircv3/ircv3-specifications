# /register

## Motivation

The lack of a standardized way to register an account has been a source of frustration for many participants
for several years.

This isn't the first attempt to fix it. Unlike the withdrawn `ACC` proposal, this effort aims to be more
appealing to implementors by being severely restricted in scope. The only register-verify flows supported
are those already in use in the wild. The authors hope that this stripped-down solution will serve as the
basis for future organic expansion.

### Capability

This specification adds the `draft/register` capability, whose presence signifies that the server
accepts the `REGISTER` command.

The capability has an optional value, a comma-separated list of key-value pairs; the format is intended
to follow the precedent set by [the multiline draft][multiline].

The `draft/register` capability SHOULD NOT be requested. Servers MAY `NAK` any such request. The
`REGISTER` command MUST be accepted regardless of any attempt to negotiate or disable the capability.

The defined keys are:

 * `before-connect` - if present, indicates the server supports early registration, so it
   MUST NOT use `FAIL REGISTER COMPLETE_CONNECTION_REQUIRED`
 * `email-required` - if present, registrations require a valid email address to process

### Commands

    REGISTER {<email> | "*"} <password>
    
The `REGISTER` command informs the server of a request to register an account named for the current
nick of the requestor.

The `REGISTER` command MAY be sent at any point during the connection that the client has a valid
nickname. If the client sends `REGISTER` before completing connection registration, and receives a
`FAIL REGISTER COMPLETE_CONNECTION_REQUIRED`, it SHOULD make a second attempt after it receives the
welcome message.

    VERIFY <account> <code>
    
The `VERIFY` command completes a registration that required email verification.

### Responses

    REGISTER SUCCESS <account> <message>
    VERIFY SUCCESS <account> <message>
    
Sent by the server when the `REGISTER` or `VERIFY` fully succeeded, and no
further action is needed.
Afterward, the client is assumed to be authenticated to the `<account>`
as if it used SASL.

    REGISTER VERIFICATION_REQUIRED <account> <message>
    
Sent by the server as a response to `REGISTER` when further action is required
to complete registration of the `<account>`, as explain in the human-readable `<message>`.

    FAIL REGISTER USERNAME_EXISTS <account> <message>

Sent by the server as a response to `REGISTER` when the client tried to register
while using a nick that is not available as a new account's name
(for example, because someone already registered it).

The server MUST NOT send this response before the client sent a `NICK` command.

The client MAY try registering again later.
Clients should not automatically retry immediately without changing their nickname
or waiting.

    FAIL REGISTER ALREADY_REGISTERED <account> <message>
    FAIL VERIFY ALREADY_REGISTERED <account> <message>

Sent by the server is the client is already authenticated.

    FAIL REGISTER WEAK_PASSWORD <account> <message>

Sent by the server if the password is considered too weak.

The client MAY try registering again.

    FAIL REGISTER INVALID_EMAIL <account> <message>

Sent by the server if it cannot send emails to the provided address.
Server implementations should be careful not to accidentally send this as a response
to a valid address.

    FAIL REGISTER UNACCEPTABLE_EMAIL <account> <message>

Sent by the server if the email address is valid, but not available for account registration.

    FAIL REGISTER COMPLETE_CONNECTION_REQUIRED
    FAIL VERIFY COMPLETE_CONNECTION_REQUIRED

Sent by the server if the client sent a `REGISTER` command before connection registration.
The server MUST NOT sent these replies if it advertizes the `before-connect` key of the
`draft/register` capability.

[multiline]: https://github.com/ircv3/ircv3-specifications/pull/398/
