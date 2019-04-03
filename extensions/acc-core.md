---
title: "The ACC command framework"
layout: spec
copyrights:
  -
    name: "William Pitcock"
    period: "2014-2015"
    email: "nenolod@dereferenced.org"
  -
    name: "Daniel Oaks"
    period: "2016-2019"
    email: "daniel@danieloaks.net"
---
## Notes for implementing work-in-progress version
This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `acc` capability name. Instead, implementations SHOULD use the `draft/acc` capability name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.


## Introduction
The `ACC` command framework provides a unified, standardized interface for account management. Typically, managing this account has been done through `NickServ` or other services bots, which has made it difficult for clients to present clean, consistent interfaces to users.

This specification defines the `ACC REGISTER` and `VERIFY` subcommands, which standardise account creation and validation to the network's authentication layer.  This allows for a network to provide a common interface regardless of what authentication layer they have chosen for their network to operate, i.e. traditional services, a different central authority, or a decentralized model similar to [IRCX](https://en.wikipedia.org/wiki/IRCX).
   
We believe that the benefit of adding a common interface for account management will result in more user-focused interfaces for common account tasks including registration (just as SASL has allowed clients to create clean account authentication interfaces). Further, standardised account registration allows client authors to provide guided account creation and validation to their users, regardless of how the network performs these tasks.


## The `draft/acc` capability
This capability indicates that the server supports the `ACC` command as described by this specification, and can use `ACC LS` to get a list of supported subcommands. Clients MAY request the `draft/acc` capability, but it will not affect whether or not they can use the command.


## The `ACC` command
The `ACC` command exposes a number of subcommands as described below. Servers MAY send `ACC` messages to clients in response to client commands, as described by the specifications describing those commands.

Servers MUST use the [standard replies extension](https://github.com/ircv3/ircv3-specifications/pull/357) to notify the client of failed `ACC` subcommands. The specific codes and format to use are given with each command's description.


## The `ACC LS` subcommand
The `ACC LS` subcommand signals that a client wishes to receive a list of supported `ACC` subcommands. The response is one or more `ACC LS` messages sent to the client with this format:

    :<server> ACC LS :subcommand names go here

If the reply contains multiple lines (due to IRC line length limitations), all but the last reply MUST have a parameter containing only an asterisk (`*`) preceding the subcommand list. This lets clients know that more `ACC LS` lines are incoming. For example:

    :<server> ACC LS * :LS REGISTER VERIFY
    :<server> ACC LS * :subcommand names go here
    :<server> ACC LS * :subcommand names go here
    :<server> ACC LS :final names go here

If this subcommand is given with any additional parameters, servers MUST ignore these and process the subcommand as normal. This allows for future extension of this subcommand.


## The `ACC REGISTER` subcommand
The `ACC REGISTER` subcommand signals the intent of a client to register an account with the network's authentication layer.  It is similar to current methods of signalling that intent.

An `ACC REGISTER` subcommand has the following format:

    ACC REGISTER <accountname> [callback_namespace:]<callback> [cred_type] :<credential>

The `accountname` parameter specifies the account name the client wishes to register.  If the account name is `*`, it indicates that the client wishes to register with their current nickname as the account name.  If the `REGNICK` ISUPPORT token is advertised, the client MUST send `*` for this parameter.

The `callback` parameter indicates where the verification code should be sent. This parameter MAY contain an asterisk `*`, indicating that no verification code is required and that account registration shall complete immediately. Otherwise, this parameter contains a standard URI, the namespace being contained in the IANA URI Schemes registry (as per [RFC 7595](https://tools.ietf.org/html/rfc7595)) and listed in the `REGCALLBACKS` ISUPPORT token. If a callback namespace is not explicitly provided, the IRC server MUST use `mailto` as a default.

The `cred_type` parameter indicates the type of authentication that's required to login to the account once it has been registered. The supported credential types are:

- `passphrase`: `<credential>` is a plain-text passphrase that the client will use to authenticate with SASL once the account is registered. Passphrases MAY contain spaces.
- `certfp`: The client's TLS certificate is used as the as the authentication mechanism for future connections. In short, if the client connects and presents the same TLS certificate, they can use the `SASL EXTERNAL` method to authenticate. The client SHOULD provide an empty credential (`*`), and the server MUST ignore the credential value provided by the client. If no certificate is presented by the client, the server MUST respond with the `REG_INVALID_CRED_TYPE` `FAIL` code.

If the client does not provide the `cred_type` parameter, the server MUST use the `passphrase` credential type. If the client provides a credential type that's now supported, the server MUST respond with the `REG_INVALID_CRED_TYPE` `FAIL` code.

Clients SHOULD NOT be sent legacy (e.g. from NickServ) privmsgs or notices in response to `ACC` commands where numerics and messages defined in this document can replace them.

After this, if verification is required, the IRC server MUST send the `RPL_REG_VERIFICATION_REQUIRED` numeric. If verification is not required, the IRC server MUST instead send the `RPL_REG_SUCCESS` numeric.

| No. | Label | Format |
| --- | ----- | ------ |
| 920 | `RPL_REG_SUCCESS` | `:<server> 920 <user_nickname> <accountname> :Account created` |
| 927 | `RPL_REG_VERIFICATION_REQUIRED` | `:<server> 927 <user_nickname> <accountname> <callback_namespace:><callback> :A verification token was sent` |

Upon error, the IRC server MUST send a [`FAIL` message](https://github.com/ircv3/ircv3-specifications/pull/357) with one of the codes below using the given format, and including an appropriate description of the error:

| Code | Format |
| ---- | ------ |
| `ACCOUNT_ALREADY_EXISTS` | `:<server> FAIL ACC ACCOUNT_ALREADY_EXISTS <accountname> :Account already exists` |
| `REG_INVALID_CALLBACK` | `:<server> FAIL ACC REG_INVALID_CALLBACK <accountname> <callback> :Callback is invalid` |
| `REG_INVALID_CRED_TYPE` | `:<server> FAIL ACC REG_INVALID_CRED_TYPE <accountname> <cred_type> :Credential type is invalid` |
| `REG_MUST_USE_REGNICK` | `:<server> FAIL ACC REG_MUST_USE_REGNICK <accountname> :Must register with current nickname instead of separate account name` |
| `REG_UNAVAILABLE` | `:<server> FAIL ACC REG_UNAVAILABLE :Account registration is currently unavailable` |
| `REG_UNSPECIFIED_ERROR` | `:<server> FAIL ACC REG_UNSPECIFIED_ERROR <accountname> [<contexts>...] <description>` |


## The `ACC VERIFY` subcommand
The `ACC VERIFY` subcommand lets a client submit a verification token to complete their account registration.  This token is one that the server has sent to the user out-of-band with the given callback namespace and method.

A `ACC VERIFY` subcommand consists of the following format:

    ACC VERIFY <accountname> <auth_code>

Upon success, the IRC server MUST send the `RPL_VERIFYSUCCESS` numeric, which looks like:

| No. | Label | Format |
| --- | ----- | ------ |
| 923 | `RPL_VERIFYSUCCESS` | `:<server> 923 <user_nickname> <accountname> :Account verification successful` |

The IRC server MUST also send an `RPL_LOGGEDIN` (900) numeric and consider the client to be logged in to the account that has been successfully verified.

Upon error, the IRC server MUST send a [`FAIL` message](https://github.com/ircv3/ircv3-specifications/pull/357) with one of the codes below using the given format, and including an appropriate description of the error:

| Code | Format |
| `ACCOUNT_ALREADY_VERIFIED` | `:<server> FAIL ACC ACCOUNT_ALREADY_VERIFIED <accountname> :Account already verified` |
| `ACCOUNT_INVALID_VERIFY_CODE` | `:<server> FAIL ACC ACCOUNT_INVALID_VERIFY_CODE <accountname> :Invalid verification code` |
| `VERIFY_UNSPECIFIED_ERROR` | `:<server> FAIL ACC VERIFY_UNSPECIFIED_ERROR <accountname> [<contexts>...] <description>` |


## The `REGCALLBACKS` `RPL_ISUPPORT` token
Implementations which require a verification token should list the types of verification callbacks they support, using the `REGCALLBACKS` RPL_ISUPPORT (005) token.  The `REGCALLBACKS` token takes a comma-separated list of supported callback methods.  In general, the `REGCALLBACKS` token is limited to URI schemes defined in the IANA URI Schemes registry, per [RFC 7595][https://tools.ietf.org/html/rfc7595].

If verification is not required by the implementation and account registration can complete immediately, the `REGCALLBACKS` token MUST contain an asterisk (`*`) which indicates this.

An example 005 reply indicating that the `mailto` and `sms` callbacks are supported is:

    :irc.example.com 005 kaniini REGCALLBACKS=mailto,sms :are supported by this server

An example 005 reply indicating that no verification callback methods are supported is:

    :irc.example.com 005 kaniini REGCALLBACKS=* :are supported by this server


## The `REGCREDTYPES` `RPL_ISUPPORT` token
Implementations which provide support for account registration using this framework MUST specify the supported credential types using the `REGCREDTYPES` RPL_ISUPPORT (005) token.  The `REGCREDTYPES` token takes a comma-separated list of supported commands.  Other extensions MAY define new credential types.

An example 005 reply indicating support for credential types is:

    :irc.example.com 005 kaniini REGCREDTYPES=passphrase,certfp :are supported by this server

The credential types specified by this document are described above in the <a href=""><code>ACC REGISTER</code> subcommand description</a>.


## The `REGNICK` `RPL_ISUPPORT` token
Some implementations restrict the account name that can be registered to the current nickname of the client (that is, they tie account name registration to nicknames). Implementations which restrict registration in this way MUST advertise the `REGNICK` RPL_ISUPPORT (005) token.

This token has no value, and an example 005 reply indicating this behaviour is:

    :irc.example.com 005 dan REGNICK :are supported by this server

When registering on a network that enforces this policy, clients MUST send `*` as `<accountname>` in the `ACC REGISTER` command (which indicates that the server use the client's current nickname as the account name). If a client sends does not send an asterisk `*` on a network enforcing this policy, the server MUST return the `REG_MUST_USE_REGNICK` `FAIL` code with an appropriate description.


## Account Required

Certain actions on a network may require a client to be logged-into an account. These may include registering a channel, joining specific channels, connecting to the network at all, etc. The `ACC_REQUIRED` `FAIL` code indicates that the client must be logged into an account in order to complete an action or continue using the network.

Upon receiving the `ACC_REQUIRED` code, clients SHOULD alert users that they need to register/login or provide an interface to do so. The `ACC_REQUIRED` code MAY be received at any time – before, during, or after connection registration.

| Code | Format |
| ---- | ------ |
| `ACC_REQUIRED` | `:<server> FAIL * ACC_REQUIRED :You must have an account to connect to this server` |

If the response was triggered by a command, this command should be given with the `FAIL` message. If the user must register outside IRC (for example, through a web interface), a link to do so should be provided in the description.


## Examples

These examples show how to perform common tasks with this framework.

### Listing the supported subcommands

    C: ACC LS
    S: ACC LS * :REGISTER VERIFY
    S: ACC LS :MFA RESETPASS

### Listing the supported subcommands with extra parameters

    C: ACC LS EXTRA_PARAMETER EXTENSION_PARAM
    S: ACC LS * :REGISTER VERIFY
    S: ACC LS :MFA RESETPASS

### Registering the account "kaniini" with e-mail address kaniini@example.com:

    C: ACC REGISTER kaniini mailto:kaniini@example.com passphrase :testpassphrase123
    S: 920 kaniini kaniini :Account registered
    S: 927 kaniini kaniini kaniini@example.com :A verification code was sent
    C: ACC VERIFY kaniini 3qw4tq4te4gf34
    S: 923 kaniini kaniini :Account verification successful
    S: 903 kaniini :Authentication successful

### Registering the account "kaniini" with e-mail address kaniini@example.com:

    C: ACC REGISTER kaniini kaniini@example.com passphrase :testpassphrase123
    S: 920 kaniini kaniini :Account registered
    S: 927 kaniini kaniini kaniini@example.com :A verification code was sent
    ...

### Registering with the client's current nickname "dan-":

    C: ACC REGISTER * mailto:dan@example.com passphrase :testpassphrase123
    S: 920 dan- dan- :Account registered
    S: 927 dan- dan- dan@example.com :A verification code was sent
    ...

### Registering the account "rabbit" with SMS number +11234567890:

    C: ACC REGISTER rabbit sms:+11234567890 passphrase :testpassphrase123
    S: 920 kaniini rabbit :Account registered
    S: 927 kaniini rabbit +11234567890 :A verification code was sent
    C: ACC VERIFY rabbit 123456
    S: 923 kaniini rabbit :Account verification successful
    S: 903 kaniini :Authentication successful

### Registering the account "rabbit" with an unsupported verification method:

    C: ACC REGISTER rabbit 1vBjNBdhjWFFbbbbVBHJEWBHJWcfbbvjkhbea passphrase :testpassphrase123
    S: FAIL ACC REG_INVALID_CALLBACK rabbit 1vBjNBdhjWFFbbbbVBHJEWBHJWcfbbvjkhbea :Callback token is not valid

### Registering the account "rabbit", but providing an invalid verification token:

    C: ACC REGISTER rabbit mailto:kaniini@example.com passphrase :testpassphrase123
    S: 920 kaniini rabbit :Account registered
    S: 927 kaniini rabbit kaniini@example.com :A verification code was sent
    C: ACC VERIFY rabbit 3qw4tq4te4gf34
    S: FAIL ACC ACCOUNT_INVALID_VERIFY_CODE rabbit :Invalid verification code

### Registering the account "rabbit" where a verification token is not required:

    C: ACC REGISTER rabbit * passphrase :testpassphrase123
    S: 920 kaniini rabbit :Account registered
    S: 903 kaniini :Authentication successful

### Registering the account "harold" where REGNICK is present in RPL_ISUPPORT:

    C: ACC REGISTER harold * passphrase :testpassphrase123
    S: FAIL ACC REG_MUST_USE_REGNICK harold :Must register with current nickname instead of separate account name

### Registering an account which already exists:

    C: ACC REGISTER rabbit sms:+11234567890 passphrase :testpassphrase123
    S: FAIL ACC ACCOUNT_ALREADY_EXISTS rabbit :Account already exists

### Registering an account with an unsupported credential type:

    C: ACC REGISTER rabbit mailto:kaniini@example.com some_invalid_cred_type :1QXvcnFWJKFGjbnkwawFJKNJKEc254
    S: FAIL ACC REG_INVALID_CRED_TYPE rabbit some_invalid_cred_type :Credential type is invalid

### Implementation-specific registration errors:

    C: ACC REGISTER rabbit sms:+11234567890 passphrase :abc123
    S: FAIL ACC REG_UNSPECIFIED_ERROR rabbit :Invalid params: "rabbit" is a restricted account name
