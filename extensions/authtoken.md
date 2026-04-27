---
title: "Authentication Tokens"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Catherine 'whitequark'"
    period: "2026"
    email: "whitequark@whitequark.org"
  -
    name: "Ryan Schmidt"
    period: "2026"
    email: "moonmoon@libera.chat"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `authtoken` capability name, `AUTHTOKEN` ISUPPORT name, or `authtoken` batch type. Instead, implementations SHOULD use the `draft/authtoken` capability name, `draft/AUTHTOKEN` ISUPPORT name, and `draft/authtoken` batch type to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name, ISUPPORT name, and batch type.

## Introduction

This specification introduces the ability for clients to request authentication tokens which may be passed to other services, and for those services to be able to validate said tokens and retrieve a list of pre-configured authorization claims for the token.

There are two primary issues surrounding an authentication layer meant to be interoperable among many parties: security and well-defined APIs that each party is able to use to operate on tokens. This specification aims to solve these challenges by defining both the means of generating and validating a token, as well as a workflow that works best with the most secure token type (single-use OTPs) but can be extended for stateless servers to support something else instead (such as JWT). Generated tokens are associated with any number of claims and are tied to one specific pre-defined (by network opers) external service to prevent token confusion attacks wherein a token generated for one service is passed to a different one to exploit differing sets of per-service claims. This is meant to be a framework that other specifications as well as custom per-network services can use to handle authentication and authorization decisions for external services in a well-defined and consistent manner.

This specification makes no assumptions that the external service is run by the same people who run the network vs. a 3rd party. It is also not tied exclusively to HTTP, although generated tokens are certainly usable there via e.g. the Authorization header. Token validation runs through the same IRC endpoint that clients use to avoid the need for networks to deploy additional services specifically for this specification.

## Dependencies

This specification depends on the [`batch`][] capability which MUST be negotiated to use the `TOKEN` command. This specification additionally defines the `draft/authtoken` capability. The order of capability negotiation is not significant and MUST NOT be enforced.

This specification additionally makes use of the [standard replies][] and [client-initiated batch][] frameworks.

## Capability

If a client negotiates the `draft/authtoken` capability prior to completing registration with the network, the output of the `TOKEN SERVICELIST` subcommand MUST be attached to the registration burst any time after sending ISUPPORT and before sending any LUSERS or MOTD output.

Clients who negotiate the `draft/authtoken` capability MUST additionally be notified on any changes to the service list. These notifications take the form `TOKEN NEW <service key> <url>` and `TOKEN DEL <service key>`. Servers MAY omit `TOKEN DEL` if an existing service is changing its URL (sending only `TOKEN NEW` with the updated URL).

Servers MUST NOT require that clients negotiate the `draft/authtoken` capability before making use of the `TOKEN` command.

## ISUPPORT token

This specification defines one ISUPPORT token. Servers MUST publish the `draft/AUTHTOKEN` ISUPPORT token to indicate they support this specification. The token does not have any value, but clients MUST NOT reject tokens published with a value, to allow for future revisions to this specification. Servers SHOULD additionally support [extended ISUPPORT][] since token verification can happen before connection registration is complete, so clients can determine the implementation status of this extension before sending any TOKEN commands.

## `draft/authtoken` batch type

The `draft/authtoken` batch type has two parameters: the service key and the URL corresponding to that service key.

When sent by a server, this batch MUST contain only `TOKEN` messages.

A client MUST negotiate the `draft/authtoken` capability before sending client-initiated `draft/authtoken` batches. A client-initiated `draft/authtoken` batch MUST contain only `TOKEN VALIDATE` commands, and these commands MUST omit the optional service and url parameters.

## TOKEN command

This specification defines one new command named TOKEN. Clients do not need to negotiate any capabilities in order to use this command. The command has three subcommands, which are defined as follows:

```
TOKEN SERVICELIST
TOKEN GENERATE <service> [<scope>]
TOKEN VALIDATE [<service> <url>] :<token>
```

The `TOKEN VALIDATE` subcommand MUST be usable before a client has completed connection registration. Servers MAY choose to allow the use of the other `TOKEN` subcommands before clients complete connection registration as well.

If a client sends an unrecognized subcommand, the server MUST reply with the following `FAIL` message:

```
FAIL TOKEN UNKNOWN_COMMAND <command> :No such subcommand TOKEN <command>
```

The text of the final parameter MAY vary wildly between implementations.

### SERVICELIST subcommand

Syntax:

```
TOKEN SERVICELIST
```

The `TOKEN SERVICELIST` subcommand provides a list of all recognized service keys as well as their corresponding URLs. The description parameter provides some sort of description about the service. This command provides the main discoverability mechanism for which services are defined on the network, avoiding the need for per-service ISUPPORT tokens or informational capabilities.

When receiving the `TOKEN SERVICELIST` command or providing an automatic service listing during a registration burst, the server MUST reply with a list of one or more services inside of a `draft/authtoken` batch. Both parameters in the `BATCH` message MUST be set to asterisk (`*`). If no services are defined, the server MUST reply with a `NOTE TOKEN NO_SERVICES` message.

Each service is a `TOKEN` message with the following syntax:

```
TOKEN SERVICE <service> <url> :<description>
```

URLs MUST be 250 bytes in length or less, in order to ensure adequate space within the IRC protocol line for the description parameter in this message and the token parameter on other `TOKEN` messages for single-line tokens. The description parameter is a textual description of the service as defined by the server's operators.

*[[Begin non-normative example--*

A user sends `TOKEN SERVICELIST` to list all services and receives the following output. One line is for the draft FILEHOST service and the other is for a user-defined QDB service under the vendor "example.com" namespace.

```
BATCH +a draft/authtoken * *
@batch=a TOKEN SERVICE FILEHOST https://upload.example.com :file upload service
@batch=a TOKEN SERVICE example.com/QDB https://qdb.example.com :Quote Database
BATCH -a
```

A user sends `TOKEN SERVICELIST` on a network where no services are configured. As such, they receive a note that no services are defined.

```
NOTE TOKEN NO_SERVICES :No services are defined for this network.
```

*--End non-normative example]]*

### GENERATE subcommand

Syntax:

```
TOKEN GENERATE <service> [<scope>]
```

When a client sends the `TOKEN GENERATE` subcommand, the server responds with an opaque authentication token usable to authenticate to the provided external service. The scope parameter indicates the scope (e.g. user or channel) that the token SHOULD be scoped to, if relevant. If no scope is needed for a particular external service, the parameter MAY be omitted. See the "Server implementation considerations" section below for recommendations about security of generated tokens.

Each line of output is a `TOKEN` message with the following syntax:

```
TOKEN GENERATE <service> <url> :<token>
```

If the reply contains multiple lines (due to IRC line length limitations), the server MUST batch the reply in a `draft/authtoken` batch. The batch MUST have two parameters: the service and URL. `TOKEN GENERATE` messages inside of the batch SHOULD use asterisks (`*`) for the service and url parameters instead of specifying them again for each batch line. When receiving a batched reply, clients MUST concatenate the token parameters together without any separators to form the complete token. If a server does not support the draft/authtoken capability, it MUST NOT generate tokens that require multiple lines.

Clients MUST NOT assume that the returned token has any particular format. Clients SHOULD re-run `TOKEN GENERATE` each time they need to interact with the external service, as there is no guarantee that a token is usable more than once.

If the server refuses to generate a token for any reason, it MUST send a `FAIL` message to the client indicating the reason a token could not be generated; see the standard replies section below for a list of potential responses. The command in the message is `TOKEN`.

*[[Begin non-normative example--*

Simple example when a client sends `TOKEN GENERATE FILEHOST #chat`:

```
TOKEN GENERATE FILEHOST https://upload.example.com :54a333f6e0c218382ab5f64c1135f07798d0255e87115c451ba67cc440f5ea84
```

Example of a token spanning multiple lines when a client sends `TOKEN GENERATE FILEHOST #chat`:

```
BATCH +12345 draft/authtoken FILEHOST https://upload.example.com
@batch=12345 TOKEN GENERATE * * :eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJodHRwczovL3VwbG9hZC5leGFtc
@batch=12345 TOKEN GENERATE * * :GxlLmNvbSIsInN1YiI6Im1vb25tb29uIiwiYWNjb3VudCI6Im1vb25tb29uIiwic2VydmljZSI
@batch=12345 TOKEN GENERATE * * :6IkZJTEVIT1NUIiwidGFyZ2V0IjoiI2NoYXQiLCJtZW1iZXJfb2YiOlsiI2NoYXQiLCIjaGVsc
@batch=12345 TOKEN GENERATE * * :CIsIiNzZWNyZXQtZnJpZW5kLWNoYW5uZWwiXSwib3BlcmF0b3Jfb2YiOlsiI2NoYXQiLCIjY2h
@batch=12345 TOKEN GENERATE * * :hdC1vcHMiLCIjaGVscCJdLCJvcGVyIjpmYWxzZSwicXVvdGEiOnsicGVyX3VwbG9hZCI6MTA0O
@batch=12345 TOKEN GENERATE * * :DU3NjAwLCJ0b3RhbCI6MTYxMDYxMjczNn0sImlhdCI6MTc3NjAxNjU3NCwiZXhwIjoxNzc2MDE
@batch=12345 TOKEN GENERATE * * :3NDc0fQ.wZi8okF7t1Ghk1jHLFyqhYwRZU4T3zG1B_qcCOz8gesMCbLhntccDoxDUgsV4OKHfQ
@batch=12345 TOKEN GENERATE * * :3vjB2dpyD2gHyXcE0JJg
BATCH -12345
```

*--End non-normative example]]*

### VALIDATE subcommand (unbatched)

Syntax:

```
TOKEN VALIDATE <service> <url> :<token>
```

An external service may connect to the IRC server in order to validate a token by issuing the `TOKEN VALIDATE` subcommand. Since this command can be sent pre-registration, it does not need to complete the full connection registration process in order to validate a token and retrieve the claims associated with it.

Servers MAY choose to implement additional restrictions before accepting a `TOKEN VALIDATE` command from a client, such as requiring the client connect from a specific IP range or authenticate with `PASS` or a TLS certificate fingerprint, in order to ensure that the client validating the token positively belongs to the service the token is associated with. If a server does this and the client is not allowed to validate the given token for the given service, the server MUST fail the request with a `FAIL TOKEN NO_PERMISSIONS <service>` message.

`TOKEN VALIDATE` commands issued outside of a `draft/authtoken` batch MUST contain the service key and URL parameters.

Servers MUST validate that the service and url parameter supplied in either the `draft/authtoken` client-initiated batch or the `TOKEN VALIDATE` command match the service and url associated with the provided token at the time it was generated. If this validation fails, servers MUST reply with `FAIL TOKEN INVALID_TOKEN` instead of producing a list of claims, even if the token is otherwise valid.

When receiving the `TOKEN VALIDATE` command, if the token is valid and associated with the provided service key and URL, the server MUST reply with a list of zero or more claims for that token inside of a `draft/authtoken` batch with two parameters: the service key and the service's URL. Each claim is a `TOKEN` message with the following syntax:

```
TOKEN CLAIM <key> :<value>
```

The key is a claim key (see below) and the value is the value of that claim. If a claim key is repeated across multiple lines, clients MUST concatenate all values for a given key together without any separators and treat the combined result as the true value for the key.

*[[Begin non-normative example--*

Response to the client sending a single `TOKEN VALIDATE` command. The server produces a leading space in the second line of the member_of claim because the client must concatenate the lines together with no separators.

```
Client:
TOKEN VALIDATE FILEHOST https://upload.example.com :54a333f6e0c218382ab5f64c1135f07798d0255e87115c451ba67cc440f5ea84

Server:
BATCH +12345 draft/authtoken FILEHOST https://upload.example.com
@batch=12345 TOKEN CLAIM account :moonmoon
@batch=12345 TOKEN CLAIM member_of :#chat #help #channel1 #channel2 #channel3
@batch=12345 TOKEN CLAIM member_of : #channel4 #channel5
@batch=12345 TOKEN CLAIM operator_of :#chat #chat-ops
@batch=12345 TOKEN CLAIM scope :#chat
BATCH -12345
```

### VALIDATE subcommand (batched)

Syntax:

```
BATCH +<id> draft/authtoken <service> <url>
@batch=<id> TOKEN VALIDATE :<token>
(send additional TOKEN VALIDATE lines as necessary until the full token is sent)
BATCH -<id>
```

If a non-batched `TOKEN VALIDATE` command exceeds IRC line length protocol limits, the batched form described here MUST be used instead. Before sending a batched `TOKEN VALIDATE`, clients MUST first negotiate the draft/authtoken capability. This variation omits the service and URL parameters from the `TOKEN VALIDATE` command as they are present in the `BATCH` command instead. The token parameter for each message inside of the batch SHOULD be no longer than 400 bytes, to prevent issues with server-side rejection or truncation of overlong messages.

The reply to a batched `TOKEN VALIDATE` command is equivalent to that of a non-batched `TOKEN VALIDATE` command, and all of the other considerations of non-batched `TOKEN VALIDATE` commands apply to batched ones as well, such as servers validating the service and URL parameters and potentially requiring some level of authentication before accepting the command.

Clients MUST NOT send batched `TOKEN VALIDATE` commands for tokens that are 200 bytes or less in length.

Example of a client sending a multiline token.

```
Client:
BATCH +a draft/authtoken FILEHOST https://upload.example.com
@batch=a TOKEN VALIDATE :eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJodHRwczovL3VwbG9hZC5leGFtc
@batch=a TOKEN VALIDATE :GxlLmNvbSIsInN1YiI6Im1vb25tb29uIiwiYWNjb3VudCI6Im1vb25tb29uIiwic2VydmljZSI
@batch=a TOKEN VALIDATE :6IkZJTEVIT1NUIiwidGFyZ2V0IjoiI2NoYXQiLCJtZW1iZXJfb2YiOlsiI2NoYXQiLCIjaGVsc
@batch=a TOKEN VALIDATE :CIsIiNzZWNyZXQtZnJpZW5kLWNoYW5uZWwiXSwib3BlcmF0b3Jfb2YiOlsiI2NoYXQiLCIjY2h
@batch=a TOKEN VALIDATE :hdC1vcHMiLCIjaGVscCJdLCJvcGVyIjpmYWxzZSwicXVvdGEiOnsicGVyX3VwbG9hZCI6MTA0O
@batch=a TOKEN VALIDATE :DU3NjAwLCJ0b3RhbCI6MTYxMDYxMjczNn0sImlhdCI6MTc3NjAxNjU3NCwiZXhwIjoxNzc2MDE
@batch=a TOKEN VALIDATE :3NDc0fQ.wZi8okF7t1Ghk1jHLFyqhYwRZU4T3zG1B_qcCOz8gesMCbLhntccDoxDUgsV4OKHfQ
@batch=a TOKEN VALIDATE :3vjB2dpyD2gHyXcE0JJg
BATCH -a
```

*--End non-normative example]]*

## Service keys

Service keys are specified by IRCv3 extensions; vendor-specific types MUST be prefixed the same way as how vendor-specific capabilities are prefixed. Custom user-defined services provided by networks SHOULD NOT use unprefixed names for their service keys. See [capability negotiation][] for the exact details.

The full service key MUST be treated as a case-insensitive identifier.

## Claim keys

Claim keys are specified by IRCv3 extensions, vendor-specific types MUST be prefixed the same way as how vendor-specific capabilities are prefixed.

The full claim key MUST be treated as an opaque identifier.

The following keys are defined:

- `account`: The user's account name.
- `member_of`: A space-separated list of channels relevant to the external service this user is a member of.
- `name`: The user's current nickname.
- `operator_of`: A space-separated list of channels relevant to the external service this user has elevated privileges in.
- `role`: A space-separated list of service-defined role names this user belongs to.
- `scope`: The scope (e.g. user or channel) that this token is valid for.

## Server implementation considerations

This section is non-normative.

For maximum security, generated tokens should be single-use and have short expiration periods (10-15 minutes) before they can no longer be validated. Providing a random string as the token is sufficient for this, instead of some other format where claims are embedded directly within the token. When listing the claims for such a random string token in the reply to `TOKEN VERIFY`, evaluating things such as channel access at the time `TOKEN VERIFY` is sent rather than caching the original value may make sense so that events such as temporary opping or split-riding can be used to elevate privileges with an external service. Alternatively, make use of the ACLs in the services database rather than actual current op access on a channel to derive permissions-based claims.

For tokens that carry claim data, such as JWTs, the expiration period should similarly be short. For servers which carry state data regarding generated tokens, the "jti" claim for JWTs (or a similar type of claim for other token types) should be used to prevent token re-use. All such tokens containing claim data should be signed. For tokens where interactive validation is expected, signing with a secret key is sufficient as the key does not need to be shared between multiple parties. For tokens where non-interactive validation is possible or expected, signing with public key encryption is preferable to avoid sharing secrets between multiple parties and so other parties cannot spoof signed tokens. If signing is employed with shared secrets despite the previous advice, a different shared secret should be used per service.

## External service implementation considerations

This section is non-normative.

This specification defines a validation flow wherein the external service receives a token via some means from a client, connects to an IRC port, and issues a `TOKEN VALIDATE` command to transform that token into a list of claims. This is robust and works with all token types potentially generated by `TOKEN GENERATE` (including claims-bearing tokens such as JWTs), and allows the external service to delegate token validity checking to the IRC server (allowing for token revocation to occur and avoiding other potential causes of security issues).

If a service chooses to validate tokens itself rather than using `TOKEN VALIDATE`, it should keep all of the following in mind to prevent security issues:

- The service must validate that the token is intended for that service, to avoid token confusion attacks. Using URL-based validation (i.e. checking that one of the claims of the token matches the URL of the service) is a good way to do this. For JWT, this would be the "aud" claim.
- The service must validate that the token came from the IRC server it is expecting tokens to come from. For JWT, this would be the "iss" claim.
- The service must validate that the token is signed by the IRC server with a secure algorithm and that the signature is valid. Public keys used to validate signatures should be pre-shared out of band.
- The service should keep track of tokens it has already seen to prevent token replay attacks--tokens should be single-use. For JWT, this would be the "jti" claim, along with some stateful tracking of previously-seen IDs.
- The service must validate that the token has not expired. For JWT, this would be a combination of the "exp" and "nbf" claims.
- The service should validate that the token was generated recently (e.g. within the past 10-15 minutes), to discourage the use of long-lived tokens and prevent vulnerabilities from leaked tokens due to lack of ability to check revocation. For JWT, this would be the "iat" claim.

Even so, external services should prefer interactive token validation via `TOKEN VALIDATE` where possible as this allows the IRC server to do revocation checks (something not possible with non-interactive validation) and to adjust the list of claims to reflect knowledge at the time the validation happens rather than the time the token was generated.

## Standard replies

The following standard replies are defined with these parameters. The text of the final parameter is an example, and the user-readable message MAY vary wildly between implementations.

| Type   | Code               | Parameters                                                                          |
| ------ | ------------------ | ----------------------------------------------------------------------------------- |
| `FAIL` | `ACCOUNT_REQUIRED` | `:You must be logged into an account to generate a token for the <service> service` |
| `FAIL` | `INTERNAL_ERROR`   | `:The requested action could not be completed due to an internal error`             |
| `FAIL` | `INVALID_SCOPE`    | `<scope> :The provided scope is invalid`                                            |
| `FAIL` | `INVALID_TOKEN`    | `:The provided token could not be validated`                                        |
| `FAIL` | `NEED_CAPABILITY`  | `<capability> :You must have the <capability> capability before using this command` |
| `FAIL` | `NO_PERMISSIONS`   | `<scope> :You do not have permission to generate a <service> token for <scope>`     |
| `FAIL` | `NO_PERMISSIONS`   | `<service> :You do not have permission to validate <service> tokens`                |
| `FAIL` | `TIMEOUT`          | `:Timeout exceeded while waiting for the complete token to validate`                |
| `FAIL` | `UNKNOWN_COMMAND`  | `<command> :No such subcommand TOKEN <command>`                                     |
| `FAIL` | `UNKNOWN_SERVICE`  | `<service> :No external service named <service> is defined`                         |
| `NOTE` | `NO_SERVICES`      | `:No services are defined for this network`                                         |

Reference table of standard replies codes and the TOKEN subcommands that produce them:

| Code               | SERVICELIST | GENERATE | VALIDATE | Other |
| ------------------ | :---------: | :------: | :------: | :---: |
| `ACCOUNT_REQUIRED` |             | *        |          |       |
| `INTERNAL_ERROR`   | *           | *        | *        |       |
| `INVALID_SCOPE`    |             | *        |          |       |
| `INVALID_TOKEN`    |             |          | *        |       |
| `NEED_CAPABILITY`  |             | *        | *        |       |
| `NO_PERMISSIONS`   |             | *        | *        |       |
| `NO_SERVICES`      | *           |          |          |       |
| `TIMEOUT`          |             |          | *        |       |
| `UNKNOWN_COMMAND`  |             |          |          | *     |
| `UNKNOWN_SERVICE`  |             | *        |          |       |

[`batch`]: ../extensions/batch.html
[capability negotiation]: ../extensions/capability-negotiation.html
[client-initiated batch]: ../extensions/client-batch.html
[extended ISUPPORT]: ../extensions/extended-isupport.html
[standard replies]: ../extensions/standard-replies.html
