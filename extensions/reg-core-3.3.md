---
title: "The REG command framework"
layout: spec
copyrights:
  -
    name: "William Pitcock"
    period: "2014-2015"
    email: "nenolod@dereferenced.org"
---

The `REG` command framework provides a unified, standardized interface for account creation and
validation to the network's authentication layer.  This allows for a network to provide a common
interface regardless of what authentication layer they have chosen for their network to operate,
i.e. traditional services, a different central authority, or a decentralized model similar to [IRCX][ircx].

   [ircx]: http://en.wikipedia.org/wiki/IRCX
   
We believe that the benefit of adding a common interface for account creation and validation,
is helpful to the end user in discovering the capabilities offered by the authentication and
authority layer of the network.  Further, we believe it is helpful to client authors, so that
they may provide guided account creation and validation for their users, regardless of network
configuration.

## The `REG` command

The `REG` command is the main command used by the account registration subsystem.  It provides
several subcommands listed in the `REGCOMMANDS` ISUPPORT type.  The core sub-commands are listed
below:

## The `REG CREATE` sub-command

The `REG CREATE` command signals the intent of a client to register an account in the network's
authentication layer.  It is similar to current methods of signalling that intent.

A `REG CREATE` command consists of the following format:

`REG CREATE <accountname> [callback_namespace:]<callback> [cred_type] :<credential>`

The `credential` field specifies a credential of `cred_type`.  If there is no specified credential, then
the `credential` field MUST be `*`.  A passphrase type `credential` MAY have spaces in it.

The `callback` field designates an opaque value which indicates where a verification code, if any,
will be sent.  The callback field is implementation-specific, with namespaces being listed using the
`REGCALLBACKS` ISUPPORT token.  An invalid callback parameter should result in the `ERR_REG_INVALID_CALLBACK`
error.  In the event that a callback is not provided, the client SHOULD send `*` to indicate that
they are not providing a callback resource.  If a callback namespace is not explicitly provided, the IRC
server MAY choose to use `mailto` as a default.

The `cred_type` field designates a value which indicates the type of supplied credential.  The accepted
values of the `cred_type` field is implementation-specific, with credential types being listed using the
`REGCREDTYPES` ISUPPORT token.  An invalid credential type parameter should result in the `ERR_REG_INVALID_CRED_TYPE`
error.  The client SHOULD supply a `cred_type` value, however if one is not provided the IRC server SHOULD
use `passphrase` as a default credential type.

The IRC server MAY forward the `REG CREATE` command to a central authority, or process it locally. Clients
SHOULD NOT be sent unnecessary legacy (e.g. from NickServ) private messages or notices in response to `REG`
commands, where native numerics such as the ones defined in this document can replace them.

Upon success, the IRC server MUST send the `RPL_REGISTRATION_SUCCESS` numeric, which looks like:

| No. | Label                      | Format                                                         |
| --- | -------------------------- | -------------------------------------------------------------- |
| 920 | `RPL_REGISTRATION_SUCCESS` | `:<server> 920 <user_nickname> <accountname> :Account created` |

If the server requires a verification token, it MUST also reply with the `RPL_REG_VERIFICATION_REQUIRED`
numeric, which looks like:

| No. | Label                           | Format                                                                                                       |
| --- | ------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| 927 | `RPL_REG_VERIFICATION_REQUIRED` | `:<server> 927 <user_nickname> <accountname> <callback_namespace:><callback> :A verification token was sent` |

Upon error, the IRC server MUST send an error code that is relevant.  We suggest these numerics:

| No. | Label                        | Format                                                                                          |
| --- | ---------------------------- | ----------------------------------------------------------------------------------------------- |
| 921 | `ERR_ACCOUNT_ALREADY_EXISTS` | `:<server> 921 <user_nickname> <accountname> :Account already exists`                           |
| 922 | `ERR_REG_UNSPECIFIED_ERROR`  | `:<server> 922 <user_nickname> <accountname> :<descriptive_text>`                               |
| 929 | `ERR_REG_INVALID_CALLBACK`   | `:<server> 929 <user_nickname> <accountname> <callback> :Callback token is invalid`             |
| 982 | `ERR_REG_INVALID_CRED_TYPE`  | `:<server> 928 <user_nickname> <accountname> <cred_type> :Credential type is invalid`           |
| 440 | `ERR_REG_UNAVAILABLE`        | `:<server> 440 <user_nickname> <subcommand> :Authentication services are currently unavailable` |

The server MAY send additional informative text upon registration success or failure using `RPL_TEXT` or `NOTICE`.

## The `REG VERIFY` sub-command

The `REG VERIFY` command signals the intent of a client to submit a verification token for their
account to the authentication layer.  The verification token MUST be sent to a valid callback
resource as specified by the client in the `REG CREATE` command.

A `REG VERIFY` command consists of the following format:

`REG VERIFY <accountname> <auth_code>`

The IRC server MAY forward the `REG VERIFY` command to a central authority, or process it
locally.

Upon success, the IRC server MUST send the `RPL_VERIFYSUCCESS` numeric, which looks like:

| No. | Label               | Format                                                                         |
| --- | ------------------- | ------------------------------------------------------------------------------ |
| 923 | `RPL_VERIFYSUCCESS` | `:<server> 923 <user_nickname> <accountname> :Account verification successful` |

The IRC server SHOULD also send an `RPL_LOGGEDIN` (900) numeric and consider the client to be logged in to the
account that has been successfully verified.

Upon error, the IRC server MUST send an error code that is relevant.  We suggest these
numerics:

| No. | Label                             | Format                                                                   |
| --- | --------------------------------- | ------------------------------------------------------------------------ |
| 924 | `ERR_ACCOUNT_ALREADY_VERIFIED`    | `:<server> 924 <user_nickname> <accountname> :Account already verified`  |
| 925 | `ERR_ACCOUNT_INVALID_VERIFY_CODE` | `:<server> 925 <user_nickname> <accountname> :Invalid verification code` |

## The `REGCOMMANDS` `RPL_ISUPPORT` token

Implementations which provide these commands SHOULD signal their availability using the `REGCOMMANDS`
RPL_ISUPPORT (005) token.  The `REGCOMMANDS` token takes a comma-separated list of supported
commands.  Other extensions MAY include additional commands in the `REGCOMMANDS` token than those
documented here.

A sample 005 reply indicating support for these commands is:

    :irc.example.com 005 kaniini REGCOMMANDS=CREATE,VERIFY :are supported by this server

## The `REGCALLBACKS` `RPL_ISUPPORT` token

Implementations which require a verification token should list the types of verification callbacks
they support, using the `REGCALLBACKS` RPL_ISUPPORT (005) token.  The `REGCALLBACKS` token takes
a comma-separated list of supported commands.  In general, the `REGCALLBACKS` token is limited to
URI schemes defined in the IANA URI schemes registry, per [RFC 4395][rfc4395].

  [rfc3495]: https://tools.ietf.org/html/rfc4395

In the event that verification is not required by the implementation, the `REGCALLBACKS` token should
be an empty value.

A sample 005 reply indicating support for verification callbacks is:

    :irc.example.com 005 kaniini REGCALLBACKS=mailto,sms :are supported by this server

A sample 005 reply indicating that no verification callback methods are supported is:

    :irc.example.com 005 kaniini REGCALLBACKS= :are supported by this server

The first callback type SHOULD be the implementation-defined default if default callback types
are supported.

## The `REGCREDTYPES` `RPL_ISUPPORT` token

Implementations which provide support for this framework SHOULD specify the allowed credential
types using the `REGCREDTYPES` RPL_ISUPPORT (005) token.  The `REGCREDTYPES` token takes a
comma-separated list of supported commands.  Other extensions MAY include additional credential
types than those listed here.

The following credential types are defined:

  * `passphrase`: an unencrypted passphrase which is sent to the server.
  * `certfp`: the client certificate fingerprint as determined by the server if available.
    The client MAY provide an empty credential (`*`) in this case.  The server SHOULD calculate
    the credential using the client certificate fingerprint, if available.  If no client certificate
    is presented, the server SHOULD NOT advertise the availability of this credential type.

A sample 005 reply indicating support for credential types is:

    :irc.example.com 005 kaniini REGCREDTYPES=passphrase,certfp :are supported by this server

The first credential type SHOULD be the implementation-defined default if default credential types
are supported.

## Examples

### Registering the account "kaniini" with e-mail address kaniini@example.com:

    C: REG CREATE kaniini mailto:kaniini@example.com passphrase :testpassphrase123
    S: 920 kaniini kaniini :Account registered
    S: 927 kaniini kaniini kaniini@example.com :A verification code was sent
    C: REG VERIFY kaniini 3qw4tq4te4gf34
    S: 923 kaniini kaniini :Account verification successful
    S: 903 kaniini :Authentication successful

### Registering the account "rabbit" with SMS number +11234567890:

    C: REG CREATE rabbit sms:+11234567890 passphrase :testpassphrase123
    S: 920 kaniini rabbit :Account registered
    S: 927 kaniini rabbit +11234567890 :A verification code was sent
    C: REG VERIFY rabbit 123456
    S: 923 kaniini rabbit :Account verification successful
    S: 903 kaniini :Authentication successful

### Registering the account "rabbit" with an unsupported verification method:

    C: REG CREATE rabbit 1vBjNBdhjWFFbbbbVBHJEWBHJWcfbbvjkhbea passphrase :testpassphrase123
    S: 929 kaniini rabbit 1vBjNBdhjWFFbbbbVBHJEWBHJWcfbbvjkhbea :Callback token is not valid

### Registering the account "rabbit", but providing an invalid verification token:

    C: REG CREATE rabbit mailto:kaniini@example.com passphrase :testpassphrase123
    S: 920 kaniini rabbit :Account registered
    S: 927 kaniini rabbit kaniini@example.com :A verification code was sent
    C: REG VERIFY rabbit 3qw4tq4te4gf34
    S: 925 kaniini rabbit :Invalid verification code

### Registering the account "rabbit" where a verification token is not required:

    C: REG CREATE rabbit * passphrase :testpassphrase123
    S: 920 kaniini rabbit :Account registered
    S: 903 kaniini :Authentication successful

### Registering an account which already exists:

    C: REG CREATE rabbit sms:+11234567890 passphrase :testpassphrase123
    S: 921 kaniini rabbit :Account already exists

### Registering an account with an unsupported credential type:

    C: REG CREATE rabbit mailto:kaniini@example.com some_invalid_cred_type :1QXvcnFWJKFGjbnkwawFJKNJKEc254
    S: 928 kaniini rabbit some_invalid_cred_type :Credential type is invalid

### Implementation-specific registration errors:

    C: REG CREATE rabbit sms:+11234567890 passphrase :abc123
    S: 922 kaniini rabbit :Invalid params: "rabbit" is a blocked account name

