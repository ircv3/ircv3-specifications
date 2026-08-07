---
title: Account 2-Factor Auth Extension
layout: spec
work-in-progress: true
extends:
  - sasl-3.1
  - sasl-3.2
  - account-registration
copyrights:
  -
    name: "Valerie Pond"
    period: "2026"
    email: "v.a.pond@outlook.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `account-2fa` capability name. Instead, implementations SHOULD
use the `draft/account-2fa` capability name to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability
name.

---

## Introduction

This specification defines a standardised mechanism for managing two-factor
authentication (2FA) on IRC accounts. It covers:

1. **Registration** of a second-factor credential with an account.  Two
   credential types are defined:
   * `totp` - RFC 6238 Time-Based One-Time Passwords, supported by the
     vast majority of authenticator apps.
   * `external` - a TLS client-certificate fingerprint, used as the
     second factor via the standard
     [`EXTERNAL`](https://datatracker.ietf.org/doc/html/rfc4422#appendix-A)
     SASL mechanism.
2. **Listing** and **removal** of registered second-factor credentials.
3. **Enforcement** -- how servers signal that a second-factor SASL step is
   required and how that step is carried out in a normal login flow.

The command and reply structure are extensible so additional credential
types may be defined by future specifications.

## Motivation

IRC networks increasingly want to offer accounts the same level of security
as modern web services.  Historically, 2FA on IRC has been implemented in
ad-hoc, network-specific ways (e.g. services bot challenges, custom
numerics).  This specification provides a consistent interface that allows
clients to offer integrated 2FA management and authentication flows.

## Architecture

### Dependencies

This specification uses the [standard replies](standard-replies.html)
framework.

Authentication enforcement described in this specification builds on
[IRCv3.1 SASL Authentication](sasl-3.1.html), as updated by
[IRCv3.2 SASL Authentication](sasl-3.2.html).

### Capability

This specification adds the `draft/account-2fa` capability.

Servers advertising this capability accept the `2FA` command described
below.

The capability MUST be advertised with a comma-separated list of supported
credential types as its CAP value, for example:

```
CAP * LS :draft/account-2fa=totp,external
```

A capability with an empty value MUST be treated by clients as advertising
no supported types (servers SHOULD always include at least one type).

### Identifiers

`<id>` values returned by `2FA LIST` and consumed by `2FA REMOVE` are
opaque ASCII strings, MUST contain only `A-Z a-z 0-9 - _`, MUST be at
most 32 characters, MUST NOT contain whitespace, and MUST be unique per
account.

`<name>` values supplied by clients to `2FA ADD` are user-chosen labels.
They MUST contain only printable non-whitespace characters and MUST be at
most 64 bytes.  Clients that wish to display a name with embedded
whitespace (e.g. "MacBook Touch ID") SHOULD substitute an underscore or
hyphen (e.g. `MacBook_Touch_ID`).

### Commands

The `2FA` command SHALL fail with `FAIL 2FA NOT_AUTHENTICATED` when
issued by a client that has not completed a SASL exchange.

#### `2FA STATUS`

    2FA STATUS

Reports whether 2FA enforcement is currently enabled on the authenticated
account.  Replies with one of:

    NOTE 2FA ENABLED  :<description>
    NOTE 2FA DISABLED :<description>

#### `2FA LIST`

    2FA LIST

Returns the list of registered credentials for the authenticated account,
one per server reply.  See [Responses](#responses) for the reply format.

If no credentials are registered the server MUST reply with:

    NOTE 2FA NO_CREDENTIALS :<description>

#### `2FA CHALLENGE`

    2FA CHALLENGE <type>

Requests an enrolment challenge for credential type `<type>`.  The server
replies with `NOTE 2FA REGISTRATION_CHALLENGE` (see
[Responses](#responses)).  The challenge SHOULD be valid for at most 120
seconds and SHOULD be invalidated after a successful `2FA ADD` of the
same type.

#### `2FA ADD`

    2FA ADD <type> <name> <data>

Registers a new second-factor credential of `<type>` for the authenticated
account.  `<data>` is a type-specific payload as defined in
[Credential Types](#credential-types).

#### `2FA REMOVE`

    2FA REMOVE <id>

Removes the registered credential identified by `<id>`.  When 2FA
enforcement is enabled and the credential being removed is the last one
registered, the server MUST reply with
`FAIL 2FA REMOVE_LAST_CREDENTIAL`; the client must first call `2FA
DISABLE`.

#### `2FA ENABLE`

    2FA ENABLE

Enables 2FA enforcement on the account.  The server MUST reply with
`FAIL 2FA NO_CREDENTIALS` if no credentials are registered.  Once enabled,
every future SASL login MUST go through the
[Two-Factor SASL Flow](#two-factor-sasl-flow).

#### `2FA DISABLE`

    2FA DISABLE <type> <data>

Disables 2FA enforcement on the account.  `<type>` MUST be one of the
credential types currently registered for the account, and `<data>` MUST
be a valid second-factor proof of that type using the same on-the-wire
format as the corresponding step-up SASL mechanism (see [Two-Factor SASL
Flow](#two-factor-sasl-flow)).  In particular:

* For `totp`, `<data>` is the 6-digit one-time password (no Base64).
* For `external`, `<data>` is the lowercase hex SHA-256 fingerprint of
  the client's TLS certificate, exactly as it would be authenticated by
  the `EXTERNAL` SASL mechanism on the current connection.  The server
  MUST reject the request unless the connection is TLS-secured and the
  presented certificate's fingerprint matches `<data>`.

The server MUST reject the request with
`FAIL 2FA INVALID_CHALLENGE_RESPONSE` if the proof does not verify.  This
prevents disabling 2FA with only the password.

### Credential Types

#### `totp`

A TOTP credential is a shared secret used with
[RFC 6238](https://tools.ietf.org/html/rfc6238) Time-Based One-Time
Passwords.  Implementations MUST support the parameter set
algorithm `SHA1`, digits `6`, period `30s` (the standard set used by all
mainstream authenticator apps).

##### Enrolment

A client begins enrolment with `2FA CHALLENGE totp`.  The server replies:

    NOTE 2FA REGISTRATION_CHALLENGE totp <challenge-data> :<description>

`<challenge-data>` is a Base64-encoded JSON object with these fields:

| Field      | Type    | Description |
| ---------- | ------- | ----------- |
| `type`     | string  | MUST be `"totp"`. |
| `secret`   | string  | Base32-encoded shared secret (≥ 16 bytes of entropy). |
| `issuer`   | string  | Human-readable issuer (typically the network name). |
| `account`  | string  | The account name. |
| `algorithm`| string  | MUST be `"SHA1"`. |
| `digits`   | integer | MUST be `6`. |
| `period`   | integer | MUST be `30`. |
| `uri`      | string  | The full `otpauth://` URI clients can render as a QR code. |

The client provisions the secret with an authenticator (typically by
displaying the QR code from `uri`) and then sends:

    2FA ADD totp <name> <code>

where `<code>` is the 6-digit OTP currently displayed by the
authenticator.  The server MUST verify `<code>` against the secret it
issued in the matching `2FA CHALLENGE totp` reply, accepting at minimum
the current 30-second window.  Implementations SHOULD additionally
accept the immediately preceding and following windows to tolerate
clock skew.

If verification succeeds the server MUST persist the secret and reply
with `2FA ADD SUCCESS`.  The server MUST NOT persist the secret unless
the client has demonstrated possession via a successful `2FA ADD`.

#### `external`

An `external` credential is a TLS client-certificate fingerprint
associated with the account.  Step-up authentication uses the standard
[`EXTERNAL`](https://datatracker.ietf.org/doc/html/rfc4422#appendix-A)
SASL mechanism: the server simply checks that the TLS connection
presented a certificate whose fingerprint matches one of the registered
credentials.

The fingerprint algorithm is SHA-256, and the fingerprint is encoded as
a lowercase hex string (no separators) of the DER-encoded certificate.
Implementations MUST NOT rely on an alternative algorithm or encoding.

##### Enrolment

A client begins enrolment with `2FA CHALLENGE external`.  The server
replies:

    NOTE 2FA REGISTRATION_CHALLENGE external <challenge-data> :<description>

`<challenge-data>` is a Base64-encoded JSON object with these fields:

| Field         | Type   | Description |
| ------------- | ------ | ----------- |
| `type`        | string | MUST be `"external"`. |
| `algorithm`   | string | MUST be `"sha-256"`. |
| `fingerprint` | string | The lowercase hex SHA-256 fingerprint of the certificate the client presented on the current TLS connection, or the empty string if no client certificate was presented. |

The `fingerprint` field lets the client confirm the value it is about
to register without having to re-derive it locally.  Clients MAY also
register a different fingerprint -- for example to enrol a hardware
security key whose certificate is not yet bound to the connection -- by
supplying that fingerprint as `<data>` directly.

The client then sends:

    2FA ADD external <name> <fingerprint>

`<fingerprint>` is the lowercase hex SHA-256 fingerprint to register.
If `<fingerprint>` is the empty string the server MUST reject the
request with `FAIL 2FA INVALID_CREDENTIAL_DATA`.  Otherwise the server
MUST persist `<fingerprint>` as a registered credential of type
`external` and reply with `2FA ADD SUCCESS`.

A server MAY require that `<fingerprint>` matches the fingerprint of
the client certificate currently presented on the TLS connection (a
"prove possession before enrol" policy) and reject mismatches with
`FAIL 2FA INVALID_CREDENTIAL_DATA`.

### Responses

    NOTE 2FA REGISTRATION_CHALLENGE <type> <challenge-data> :<description>

Sent in response to `2FA CHALLENGE <type>`.

    NOTE 2FA CREDENTIAL <id> <type> <name> <created-at> :<description>

Sent for each registered credential in response to `2FA LIST`.

* `<id>`: opaque identifier (see [Identifiers](#identifiers)).
* `<type>`: credential type.
* `<name>`: client-supplied label (see [Identifiers](#identifiers)).
* `<created-at>`: ISO 8601 UTC timestamp of registration, e.g.
  `2026-04-28T12:00:00Z`.

If 2FA is enabled but the account has no credentials (which can happen
transiently after server-side data migrations), the server SHOULD send
no `CREDENTIAL` lines and instead send:

    NOTE 2FA NO_CREDENTIALS :<description>

The other status replies are:

    NOTE 2FA ENABLED  :<description>
    NOTE 2FA DISABLED :<description>

Successful operation replies use the standard-replies positive form:

    2FA ADD SUCCESS    <type> <id> :<description>
    2FA REMOVE SUCCESS <id>        :<description>
    2FA ENABLE SUCCESS             :<description>
    2FA DISABLE SUCCESS            :<description>

### Error Replies

All errors use the `FAIL` message from the
[standard replies](standard-replies.html) framework with command `2FA`:

| Code                          | Description |
| ----------------------------- | ----------- |
| `NOT_AUTHENTICATED`           | The client is not authenticated to an account. |
| `INVALID_TYPE`                | The `<type>` is not supported by this server. |
| `UNKNOWN_CREDENTIAL`          | The `<id>` does not correspond to a registered credential. |
| `NO_CREDENTIALS`              | `2FA ENABLE` was attempted but no credentials are registered. |
| `INVALID_CHALLENGE_RESPONSE`  | The `<data>` in `2FA DISABLE` did not pass second-factor verification. |
| `INVALID_CREDENTIAL_DATA`     | The `<data>` in `2FA ADD` could not be parsed or failed verification. |
| `INVALID_NAME`                | The `<name>` parameter contains forbidden characters. |
| `NO_CHALLENGE`                | `2FA ADD` was issued without a prior `2FA CHALLENGE`, or the prior challenge has expired. |
| `REMOVE_LAST_CREDENTIAL`      | Removing the last credential while 2FA is enabled requires `2FA DISABLE` first. |
| `ALREADY_ENABLED`             | `2FA ENABLE` was issued on an already-enabled account. |
| `ALREADY_DISABLED`            | `2FA DISABLE` was issued on an already-disabled account. |
| `RATE_LIMITED`                | The command was issued too frequently and is being throttled. |
| `TEMPORARILY_UNAVAILABLE`     | The command cannot be processed at this time. |

## Two-Factor SASL Flow

When an account has 2FA enforcement enabled, a single successful SASL
exchange (e.g. `PLAIN`, `SCRAM-SHA-256`) authenticates the *identity* but
does not complete login.  The server MUST withhold `900` and `903` after
the first-factor exchange and instead send a second-factor challenge.

### Step-Up Challenge

After the server determines that the first-factor SASL exchange would
otherwise have produced `900`/`903`, if 2FA is enforced the server
instead sends:

    AUTHENTICATE 2FA-REQUIRED

This is a synthetic `AUTHENTICATE` message issued by the server (not the
client) and is the cue for the client to begin a second-factor SASL
exchange.  The server MUST NOT send `900` or `903` until the
second-factor exchange has succeeded.

The client then begins a new SASL exchange using one of the
second-factor mechanisms below by sending `AUTHENTICATE <mech>` exactly
as it would for any other SASL mechanism.

### Account With No Registered Credentials

If 2FA enforcement is enabled on an account but no credentials are
registered (e.g. as a result of operator intervention or a data
migration), the server MUST NOT issue `AUTHENTICATE 2FA-REQUIRED`
since no second-factor mechanism can succeed.  Instead, after the
first-factor exchange would otherwise have produced `900`/`903`, the
server MUST emit:

    FAIL 2FA NO_CREDENTIALS :<description>
    :irc.example.com 904 <nick> :SASL authentication failed

Recovery requires operator intervention to clear the 2FA enforcement
flag on the account.  Servers MUST NOT silently fall back to
single-factor login in this state, as that would defeat the purpose of
2FA when the credential store has been tampered with.

### Step-up SASL Mechanisms

#### `TOTP`

The `TOTP` SASL mechanism is a single-message exchange.

```
Client: AUTHENTICATE TOTP
Server: AUTHENTICATE +
Client: AUTHENTICATE <Base64(<6-digit code>)>
```

The server Base64-decodes the client message, expects a 6-digit ASCII
decimal string, and verifies it against every registered TOTP credential
on the account, accepting at minimum the current 30-second window
(implementations SHOULD also accept ±1 window for clock skew).  On
success the server emits `900`/`903`; on failure it emits `904`.

#### `EXTERNAL`

The standard [`EXTERNAL`](https://datatracker.ietf.org/doc/html/rfc4422#appendix-A)
SASL mechanism is used unchanged as a second-factor step.  The
exchange is a single message:

```
Client: AUTHENTICATE EXTERNAL
Server: AUTHENTICATE +
Client: AUTHENTICATE +
```

The trailing `+` is the SASL "empty response" marker, indicating that
the client is requesting authentication as the same identity already
established by the first factor.

The server computes the SHA-256 fingerprint of the certificate the
client presented during the TLS handshake on the current connection
and compares it against the registered `external` credentials for the
account.  On a match the server emits `900`/`903`; on no match (or if
no client certificate was presented) the server emits `904`.

A server MUST NOT accept `EXTERNAL` as a second factor on a non-TLS
connection.

### Interaction with `CAP END`

Clients MUST NOT send `CAP END` between the first-factor and
second-factor SASL exchanges.  Servers MUST NOT allow registration to
complete between the two steps.

If the client sends `CAP END` after a successful first-factor exchange
but before completing the second-factor exchange, the server MUST abort
the partial authentication and register the client without any account
association, as if authentication had not been attempted.

### Mechanism Advertisement

Servers that have 2FA enforcement enabled for any account on the network
SHOULD advertise the second-factor mechanisms alongside first-factor
mechanisms in the `sasl=` CAP value:

```
CAP * LS :sasl=PLAIN,SCRAM-SHA-256,TOTP,EXTERNAL draft/account-2fa=totp,external
```

Doing so allows clients to pre-negotiate the second-factor mechanism if
desired.

## Examples

### Enrolling a TOTP credential

```
Client: 2FA CHALLENGE totp
Server: NOTE 2FA REGISTRATION_CHALLENGE totp <Base64(JSON)> :Scan the QR code in your authenticator app then run /2FA ADD totp <name> <code>
Client: 2FA ADD totp Phone 524891
Server: 2FA ADD SUCCESS totp cred-001 :Credential 'Phone' registered
```

### Enrolling an EXTERNAL credential

```
Client: 2FA CHALLENGE external
Server: NOTE 2FA REGISTRATION_CHALLENGE external <Base64(JSON{type:"external",algorithm:"sha-256",fingerprint:"a3f1...c9"})> :Confirm the fingerprint of your client certificate then run /2FA ADD external <name> <fingerprint>
Client: 2FA ADD external Laptop a3f1b2c4d5e6f7080910111213141516171819202122232425262728293031c9
Server: 2FA ADD SUCCESS external cred-002 :Credential 'Laptop' registered
```

### Listing credentials

```
Client: 2FA LIST
Server: NOTE 2FA CREDENTIAL cred-001 totp     Phone    2026-04-28T12:00:00Z :Registered credential
Server: NOTE 2FA CREDENTIAL cred-002 external Laptop   2026-04-28T14:32:10Z :Registered credential
```

### Enabling 2FA

```
Client: 2FA STATUS
Server: NOTE 2FA DISABLED :Two-factor authentication is not enabled
Client: 2FA ENABLE
Server: 2FA ENABLE SUCCESS :Two-factor authentication is now required for login
```

### Disabling 2FA via TOTP

```
Client: 2FA DISABLE totp 524891
Server: 2FA DISABLE SUCCESS :Two-factor authentication has been disabled
```

### Removing a credential

```
Client: 2FA REMOVE cred-002
Server: 2FA REMOVE SUCCESS cred-002 :Credential 'Laptop' removed
```

### Login with TOTP step-up

```
Client: CAP LS 302
Server: CAP * LS :sasl=PLAIN,SCRAM-SHA-256,TOTP draft/account-2fa=totp batch cap-notify
Client: CAP REQ :sasl draft/account-2fa
Server: CAP * ACK :sasl draft/account-2fa
Client: AUTHENTICATE PLAIN
Server: AUTHENTICATE +
Client: AUTHENTICATE YWxpY2UAYWxpY2UAaHVudGVyMg==
Server: AUTHENTICATE 2FA-REQUIRED
Client: AUTHENTICATE TOTP
Server: AUTHENTICATE +
Client: AUTHENTICATE NTI0ODkx
Server: :irc.example.com 900 alice alice!alice@host alice :You are now logged in as alice
Server: :irc.example.com 903 alice :SASL authentication successful
Client: CAP END
```

### Failed second factor

```
Client: AUTHENTICATE TOTP
Server: AUTHENTICATE +
Client: AUTHENTICATE MDAwMDAw
Server: :irc.example.com 904 alice :SASL authentication failed
```

## Security Considerations

* Servers MUST ensure that a `2FA DISABLE` request is accompanied by a
  valid second-factor proof before removing enforcement.  Allowing
  disablement with only a password check would defeat the purpose of
  2FA.
* Servers MUST rate-limit `2FA ADD`, `2FA REMOVE`, and `2FA DISABLE`
  commands as well as TOTP step-up attempts to prevent online
  brute-force attacks against the 6-digit OTP.  A practical
  implementation is at most 5 failed attempts per minute per account.
* TOTP secrets are sensitive and MUST be stored in a way that minimises
  exposure (e.g. encrypted at rest where the deployment supports it).
* Servers MUST NOT echo the registered TOTP secret back to the client
  outside the original `REGISTRATION_CHALLENGE` reply.  In particular,
  `2FA LIST` MUST NOT reveal the secret.
* The `AUTHENTICATE 2FA-REQUIRED` signal MUST only be issued after the
  first-factor credentials have been fully verified.  It MUST NOT be
  used to leak whether an account exists or has 2FA enabled before
  credentials are checked, in order to prevent user-enumeration attacks.
* Verification of 6-digit OTPs SHOULD use a constant-time comparison.
* Clients SHOULD display the registered credential list to the user
  before confirming removal and SHOULD warn when the user is about to
  remove their last enrolled credential.
* If 2FA enforcement is enabled but no credentials remain registered,
  the server MUST lock the account out of SASL login (see [Account With
  No Registered Credentials](#account-with-no-registered-credentials))
  rather than degrading to single-factor.  Treating credential loss as
  "2FA effectively off" would let an attacker who can write to the
  credential store bypass 2FA.

