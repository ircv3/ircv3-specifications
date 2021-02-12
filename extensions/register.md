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
    
TBC

    REGISTER VERIFICATION_REQUIRED <account> <message>
    
TBC

    FAIL REGISTER USERNAME_EXISTS <account> <message>
    FAIL REGISTER ALREADY_REGISTERED <account> <message>
    FAIL REGISTER WEAK_PASSWORD <account> <message>
    FAIL REGISTER INVALID_EMAIL <account> <message>
    FAIL REGISTER UNACCEPTABLE_EMAIL <account> <message>
    FAIL REGISTER COMPLETE_CONNECTION_REQUIRED
    FAIL VERIFY USERNAME_EXISTS <account> <message>
    FAIL VERIFY ALREADY_REGISTERED <account> <message>
    FAIL VERIFY WEAK_PASSWORD <account> <message>
    FAIL VERIFY INVALID_EMAIL <account> <message>
    FAIL VERIFY UNACCEPTABLE_EMAIL <account> <message>
    FAIL VERIFY COMPLETE_CONNECTION_REQUIRED

TBC



[multiline]: https://github.com/ircv3/ircv3-specifications/pull/398/
