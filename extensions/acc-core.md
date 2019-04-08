---
title: "The Account Management framework"
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
The Account Management framework provides a unified, standardized interface for account management using the `ACC` command. Typically, managing this account has been done through `NickServ` or other services bots, which has made it difficult for clients to present clean, consistent interfaces to users.

This specification defines the `ACC REGISTER` and `VERIFY` subcommands, which standardise account creation and validation to the network's authentication layer.  This allows for a network to provide a common interface regardless of what authentication layer they have chosen for their network to operate, i.e. traditional services, a different central authority, or a decentralized model similar to [IRCX](https://en.wikipedia.org/wiki/IRCX).
   
We believe that the benefit of adding a common interface for account management will result in more user-focused interfaces for common account tasks including registration (just as SASL has allowed clients to create clean account authentication interfaces). Further, standardised account registration allows client authors to provide guided account creation and validation to their users, regardless of how the network performs these tasks.


## The `draft/acc` capability
This capability indicates that the server supports the `ACC` command as described by this specification, and can use `ACC LS` to get a list of supported subcommands. Clients MAY request the `draft/acc` capability, but it will not affect whether or not they can use the command.


## The `ACC` command
The `ACC` command exposes a number of subcommands as described below. Servers MAY send `ACC` messages to clients in response to client commands, as described by the specifications describing those commands.

Servers MUST use the [standard replies extension](https://github.com/ircv3/ircv3-specifications/pull/357) to notify the client of failed `ACC` subcommands. The specific codes and format to use are given with each command's description.

Servers MUST allow the `ACC LS` subcommand to be used at any time (before, during, and after connection registration). Servers MAY allow clients to use other subcommands while connecting to the server (for example, the server may allow registering for or verifying an account while connecting). If the client attempts to run a disallowed subcommand before connection registration has completed, the server MUST return the `ERR_NOTREGISTERED (451)` numeric.


## The `ACC LS` subcommand
The `ACC LS` subcommand signals that a client wishes to receive a list of supported `ACC` subcommands. The response is one or more `ACC LS` messages sent to the client with this format:

    :<server> ACC LS [*] <type> <data>

And here are examples of the returned data:

    :<server> ACC LS SUBCOMMANDS :LS REGISTER VERIFY
    :<server> ACC LS CALLBACKS :* mailto sms
    :<server> ACC LS CREDTYPES :passphrase certfp
    :<server> ACC LS FLAGS :regnick

For the `SUBCOMMANDS` type, `<data>` is a space-separated, case-insensitive list of supported subcommands.

For the `CALLBACKS` type, `<data>` is a space-separated list of supported verification callback namespaces.

For the `CREDTYPES` type, `<data>` is a space-separated list of supported account credential types.

For the `FLAGS` type, `<data>` is a space-separated, case-insensitive list of flags that specify additional information about how the account system on this server works.

Callbacks, credential types, and flags, along with what they mean, are described at the end of this document.

Clients MUST silently ignore types that they do not recognise.

The presence of the asterisk (`*`) parameter indicates that the reply contains additional lines. This lets clients collect all `ACC LS` information before, for example, presenting an account registration interface to the user or taking some other action.

Here is an example:

    :<server> ACC LS * SUBCOMMANDS :LS REGISTER
    :<server> ACC LS * SUBCOMMANDS :VERIFY
    :<server> ACC LS * CALLBACKS :*
    :<server> ACC LS * CREDTYPES :passphrase certfp
    :<server> ACC LS * CREDTYPES :another-example-type
    :<server> ACC LS FLAGS :regnick

If this subcommand is given with any additional parameters, servers MUST ignore these and process the subcommand as normal. This allows for future extension of this subcommand.


## The `ACC REGISTER` subcommand
The `ACC REGISTER` subcommand signals the intent of a client to register an account with the network's authentication layer.  It is similar to current methods of signalling that intent.

An `ACC REGISTER` subcommand has the following format:

    ACC REGISTER <accountname> [callback_namespace:]<callback> [cred_type] :<credential>

The `accountname` parameter specifies the account name the client wishes to register.  If the account name is `*`, it indicates that the client wishes to register with their current nickname as the account name.  If the `regnick` flag is advertised, the client MUST send `*` for this parameter.

The `callback` parameter indicates where the verification code should be sent. This parameter MAY contain an asterisk `*`, indicating that no verification code is required and that account registration shall complete immediately. Otherwise, this parameter contains a standard URI, the namespace being contained in the IANA URI Schemes registry (as per [RFC 7595](https://tools.ietf.org/html/rfc7595)) and listed in an `ACC LS CALLBACKS` message. If a callback namespace is not explicitly provided, the IRC server MUST use `mailto` as a default.

The `cred_type` parameter indicates the type of authentication that's required to login to the account once it has been registered. The supported credential types are listed below in the <a href="#"><code>ACC LS CALLBACKS</code> section</a>.

If the client does not provide the `cred_type` parameter, the server MUST use the `passphrase` credential type. If the client provides a credential type that's not supported, the server MUST respond with the `REG_INVALID_CRED_TYPE` `FAIL` code. If the given credential is invalid (for example, the client tries to use the `certfp` method without advertising a client certificate, or tries to use `passphrase` but gives an empty passphrase parameter), the server MUST respond with the `REG_INVALID_CREDENTIAL` `FAIL` code.

Clients SHOULD NOT be sent legacy (e.g. from NickServ) privmsgs or notices in response to `ACC` commands where numerics and messages defined in this document can replace them.

After this, if verification is required, the IRC server MUST generate and send the verification token to the user out-of-band using the given callback information, and send the client the `RPL_REG_VERIFICATION_REQUIRED` numeric. The verification token MAY NOT be an asterisk (`*`), and SHOULD be an unguessable combination of characters. If verification is not required, the IRC server MUST instead send the `RPL_REG_SUCCESS` and `RPL_LOGGEDIN` (900) numerics.

| No. | Label | Format |
| --- | ----- | ------ |
| 920 | `RPL_REG_SUCCESS` | `:<server> 920 <user_nickname> <accountname> :Account created` |
| 927 | `RPL_REG_VERIFICATION_REQUIRED` | `:<server> 927 <user_nickname> <accountname> <callback_namespace:><callback> :A verification token was sent` |

Upon error, the IRC server MUST send a [`FAIL` message](https://github.com/ircv3/ircv3-specifications/pull/357) with one of the codes below using the given format, and including an appropriate description of the error:

| Code | Format |
| ---- | ------ |
| `ACCOUNT_ALREADY_EXISTS` | `:<server> FAIL ACC ACCOUNT_ALREADY_EXISTS <accountname> :Account already exists` |
| `REG_INVALID_ACCOUNT_NAME` | `:<server> FAIL ACC REG_INVALID_ACCOUNT_NAME <accountname> :Account name is invalid` |
| `REG_INVALID_CALLBACK` | `:<server> FAIL ACC REG_INVALID_CALLBACK <accountname> <callback> :Cannot send verification code there` |
| `REG_INVALID_CRED_TYPE` | `:<server> FAIL ACC REG_INVALID_CRED_TYPE <accountname> <cred_type> :Credential type is invalid` |
| `REG_INVALID_CREDENTIAL` | `:<server> FAIL ACC REG_INVALID_CREDENTIAL <accountname> :Passphrase is invalid` |
| `REG_MUST_USE_REGNICK` | `:<server> FAIL ACC REG_MUST_USE_REGNICK <accountname> :Must register with current nickname instead of separate account name` |
| `REG_UNAVAILABLE` | `:<server> FAIL ACC REG_UNAVAILABLE :Account registration is currently unavailable` |
| `REG_UNSPECIFIED_ERROR` | `:<server> FAIL ACC REG_UNSPECIFIED_ERROR <accountname> [<contexts>...] <description>` |

In particular, the `REG_INVALID_CREDENTIAL` code's description SHOULD describe why the given credential is invalid in a way that helps the user resolve the problem. For example, _"Passphrase is invalid"_, _"You must connect with a TLS client certificate to use certfp"_.


## The `ACC VERIFY` subcommand
The `ACC VERIFY` subcommand lets a client submit a verification token to complete their account registration.  This token is one that the server has sent to the user out-of-band with the given callback namespace and method.

A `ACC VERIFY` subcommand consists of the following format:

    ACC VERIFY <accountname> <auth_code>

Upon success, the IRC server MUST send the `RPL_VERIFY_SUCCESS` numeric, which looks like:

| No. | Label | Format |
| --- | ----- | ------ |
| 923 | `RPL_VERIFY_SUCCESS` | `:<server> 923 <user_nickname> <accountname> :Account verification successful` |

The IRC server MUST also send an `RPL_LOGGEDIN` (900) numeric and consider the client to be logged in to the account that has been successfully verified.

Upon error, the IRC server MUST send a [`FAIL` message](https://github.com/ircv3/ircv3-specifications/pull/357) with one of the codes below using the given format, and including an appropriate description of the error:

| Code | Format |
| ---- | ------ |
| `ACCOUNT_ALREADY_VERIFIED` | `:<server> FAIL ACC ACCOUNT_ALREADY_VERIFIED <accountname> :Account already verified` |
| `ACCOUNT_INVALID_VERIFY_CODE` | `:<server> FAIL ACC ACCOUNT_INVALID_VERIFY_CODE <accountname> :Invalid verification code` |
| `REG_UNAVAILABLE` | `:<server> FAIL ACC REG_UNAVAILABLE :Account registration is currently unavailable` |
| `VERIFY_UNSPECIFIED_ERROR` | `:<server> FAIL ACC VERIFY_UNSPECIFIED_ERROR <accountname> [<contexts>...] <description>` |

Servers MAY allow IRC Operators or other privileged users to verify a given account by supplying an asterisk (`*`) in place of the `<auth_code>` parameter. If a client does verify an account in this way, they are sent the `RPL_VERIFY_SUCCESS` numeric, but are not logged into the account or sent the `RPL_LOGGEDIN` numeric.


## ACC LS CALLBACKS
Implementations which require a verification token MUST list the verification callback types that they support. The `ACC LS CALLBACKS` data parameter is a space-separate list of callback methods. In general, these methods are limited to URI schemes defined in the IANA URI Schemes registry, per [RFC 7595][https://tools.ietf.org/html/rfc7595]. However, if verification is not required by the implementation and account registration can complete immediately with just the `ACC REGISTER` subcommand, the supported verification method list MUST contain an asterisk (`*`) which indicates this.

An example `ACC LS` reply indicating that the `mailto` and `sms` callbacks are supported:

    :irc.example.com ACC LS CALLBACKS :mailto sms

An example `ACC LS` reply indicating that no verification callbacks are supported or required:

    :irc.example.com ACC LS CALLBACKS *


## ACC LS CREDTYPES
Implementations which provide support for account registration using this framework MUST specify the credential types which they support. The `ACC LS CREDTYPES` data parameter is a space-separated list of credential types.

The credential types defined here are:

- `passphrase`: `<credential>` is a plain-text passphrase that the client will use to authenticate with SASL once the account is registered. Passphrases MAY contain spaces unless the `nospaces` flag is advertised.
- `certfp`: The client's TLS certificate is used as the as the authentication mechanism for future connections. In short, if the client connects and presents the same TLS certificate, they can use the `SASL EXTERNAL` method to authenticate. The client SHOULD provide an empty credential (`*`), and the server MUST ignore the credential value provided by the client. If no certificate is presented by the client, the server MUST respond with the `REG_INVALID_CREDENTIAL` `FAIL` code.

Other extensions MAY define new credential types.

An example `ACC LS` reply indicating that the `passphrase` and `certfp` credential types are supported:

    :irc.example.com ACC LS CREDTYPES : passphrase certfp

An example `ACC LS` reply indicating that just the `passphrase` credential type is supported:

    :irc.example.com ACC LS CREDTYPES passphrase


## ACC LS FLAGS
Some implementations have other restrictions or options for account registration and management. The `ACC LS FLAGS` data parameter is a space-separated, case-insensitive list of flags for clients.

An example `ACC LS` reply displaying the `regnick` and made-up `others` flag:

    :irc.example.com ACC LS FLAGS :regnick others

An example `ACC LS` reply displaying the `regnick` flag:

    :irc.example.com ACC LS FLAGS regnick

Clients MUST silently ignore flags that they do not understand.

### regnick Flag
The `regnick` flag indicates that when a client registers an account, the account name MUST be the current nickname of the client (that is, they tie account name registration to nicknames).

When registering an account on a network with this flag, clients MUST send an asterisk (`*`) as the `<accountname>` in the `ACC REGISTER` command. This indicates that the server use the client's current nickname as the account name. If a client sends does not send an asterisk `*` on a network enforcing this policy, the server MUST return the `REG_MUST_USE_REGNICK` `FAIL` code.

### nospaces Flag
The `nospaces` flag indicates that passphrase MAY NOT contain space characters. This may include just the SPACE (`0x20`) ASCII character or any other form of whitespace as well. When this flag is advertised, clients SHOULD NOT send a passphrase that contains any whitespace, as it may be rejected by the server with the `REG_INVALID_CREDENTIAL` `FAIL` code.


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
    S: 927 kaniini kaniini kaniini@example.com :A verification code was sent

    ... server sends an email to kaniini@example.com with the verification code ...

    C: ACC VERIFY kaniini 3qw4tq4te4gf34
    S: 923 kaniini kaniini :Account verification successful
    S: 900 kaniini kaniini!kan@ini kaniini :You are now logged in as kaniini

### Registering the account "kaniini" with e-mail address kaniini@example.com:

    C: ACC REGISTER kaniini kaniini@example.com passphrase :testpassphrase123
    S: 927 kaniini kaniini kaniini@example.com :A verification code was sent
    ...

### Registering with the client's current nickname "dan-":

    C: ACC REGISTER * mailto:dan@example.com passphrase :testpassphrase123
    S: 927 dan- dan- dan@example.com :A verification code was sent
    ...

### Registering the account "rabbit" with SMS number +11234567890:

    C: ACC REGISTER rabbit sms:+11234567890 passphrase :testpassphrase123
    S: 927 kaniini rabbit +11234567890 :A verification code was sent

    ... server sends a text message to +11234567890 with the verification code ...

    C: ACC VERIFY rabbit 123456
    S: 923 kaniini rabbit :Account verification successful
    S: 900 kaniini kaniini!kan@ini rabbit :You are now logged in as rabbit

### Registering the account "rabbit" with an unsupported verification method:

    C: ACC REGISTER rabbit 1vBjNBdhjWFFbbbbVBHJEWBHJWcfbbvjkhbea passphrase :testpassphrase123
    S: FAIL ACC REG_INVALID_CALLBACK rabbit 1vBjNBdhjWFFbbbbVBHJEWBHJWcfbbvjkhbea :Callback token is not valid

### Registering the account "rabbit", but providing an invalid verification token:

    C: ACC REGISTER rabbit mailto:kaniini@example.com passphrase :testpassphrase123
    S: 927 kaniini rabbit kaniini@example.com :A verification code was sent

    ... server sends an email to kaniini@example.com with the verification code ...

    C: ACC VERIFY rabbit 3qw4tq4te4gf34
    S: FAIL ACC ACCOUNT_INVALID_VERIFY_CODE rabbit :Invalid verification code

### Registering the account "rabbit" where a verification token is not required:

    C: ACC REGISTER rabbit * passphrase :testpassphrase123
    S: 920 kaniini rabbit :Account registered
    S: 900 kaniini kaniini!kan@ini rabbit :You are now logged in as rabbit

### Operator verifying a user's account registration:

In this example, C1 is the normal user and C2 is the IRC operator or other privileged user.

    C1: ACC REGISTER pix mailto:pix@example.com passphrase :testpassphrase123
    S1: 927 daniel pix pix@example.com :A verification code was sent

    C2: ACC VERIFY pix *
    S2: 923 george pix :Account verification successful

    ... C1 can now perform SASL authentication to log into the account ..

### Registering the account "harold" where regnick is advertised:

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
