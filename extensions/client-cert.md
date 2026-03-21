---
title: Client certificate management extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Hubert Hirtz"
    period: "2025"
    email: "hubert@hirtz.pm"
  -
    name: "Simon Ser"
    period: "2026"
    email: "contact@emersion.fr"
---

# Client certificate management

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `client-cert` CAP name and batch type. Instead, implementations
SHOULD use the `draft/client-cert` CAP name and batch type to be
interoperable with other software implementing a compatible work-in-progress
version. The final version of the specification will use unprefixed CAP name
and batch type.

## Introduction

This document describes an IRC extension that enables clients to manage TLS
client certificates that are accepted for authentication with SASL EXTERNAL.
Such certificates are called "pinned certificates".

Each pinned certificate is identified by what this document calls its
fingerprint. The fingerprint of a TLS client certificate is the SHA-512 digest
of its DER encoding as defined by [RFC 6234].

Each pinned certificate is also associated with a client-provided name that is
meant to be human-readable and shown to users.

Two kinds of authentications are to be distinguished when reading this document:

- SASL authentication using the SASL IRC extension; and
- TLS client authentication (also called TLS authentication in this document),
  using mechanisms defined in [RFC 8446].

## Architecture

This extension depends on the [SASL 3.2] extension and the [standard replies]
framework.

This extension optionally depends on the [batch] extension.

This extension defines a `draft/client-cert` capability as well as a
`CLIENTCERT` command.

## `draft/client-cert` capability

Before sending any `CLIENTCERT` command, both following conditions MUST be
fulfilled:

- the `draft/client-cert` capability is successfully negotiated,
- the client is authenticated using SASL.

## `CLIENTCERT` command

The `CLIENTCERT` command sent by the client has one of the following formats:

```
CLIENTCERT CREATE <name>
CLIENTCERT LIST
CLIENTCERT DELETE [fingerprint]
```

The `CLIENTCERT` command sent by the server has one of the following formats:

```
CLIENTCERT CREATE
CLIENTCERT LIST <fingerprint> <attributes>
CLIENTCERT DELETE [fingerprint]
```

In the above listings, the parameters have the following formats:

- `attributes` follows the format of `tag` defined by the [message-tags]
  extension.
- `fingerprint` is a hex-encoded SHA-512 digest as defined by [RFC 4648] and
  [RFC 6234].

If the server receives a subcommand other than those listed above, it MUST reply
with:

```
FAIL CLIENTCERT UNKNOWN_COMMAND <subcommand> :Unknown subcommand
```

### `CLIENTCERT CREATE` subcommand

The `CREATE` subcommand has the following format when sent by the client:

```
CLIENTCERT CREATE <name>
```

The `CREATE` subcommand has the following format when sent by the server:

```
CLIENTCERT CREATE
```

Clients that haven't authenticated using SASL EXTERNAL, and that are wishing to
authenticate through SASL EXTERNAL using the TLS authentication of the current
connection may send the `CREATE` subcommand.

If the server cannot process the create request, it MUST send a standard failure
message with one of the following formats:

- `FAIL CLIENTCERT NOCERT CREATE :...`: the server cannot authenticate the TLS
  client leaf certificate of the current TLS session, either because it is
  missing or for other reasons.
- `FAIL CLIENTCERT INTERNAL_ERROR CREATE :...`: the server was able to
  authenticate the TLS client leaf certificate of the current TLS session but is
  unable to pin it, for example due to a storage failure.

If the server successfully processed the create request, it MUST send back a
`CREATE` subcommand to the client.

The server SHOULD NOT change the name of the pinned certificate to the value of
the `name` parameter if the certificate is already pinned. This is so that
clients can send `CREATE` subcommands unconditionally and not overwrite any name
customization by previous commands.

The server MUST NOT fail on the sole reason that the `name` parameter is deemed
invalid. Instead, it MUST reply with an accepted name. The client MUST NOT
assume the pinned certificate will have the same `name` parameter in future
`LIST` replies.

Once the server sends the `CREATE` subcommand, and until the certificate
fingerprint is deleted with the `DELETE` subcommand, the server SHOULD
successfully accept SASL EXTERNAL authentication attempts with the TLS leaf
certificate. However, a client MUST NOT assume that SASL EXTERNAL authentication
attempts will succeed even after receiving the `CREATE` subcommand, since the
certificate's fingerprint might be deleted by another client concurrently.

The server MAY send the `CREATE` subcommand on their own. The client SHOULD
handle receiving unprompted `CREATE` subcommands and remember it can
authenticate using SASL EXTERNAL in future IRC sessions.

### `CLIENTCERT LIST` subcommand

When sent by clients, the `LIST` subcommand has the following format:

```
CLIENTCERT LIST
```

When sent by servers, the `LIST` subcommand has the following format:

```
CLIENTCERT LIST <fingerprint> <attributes>
```

Clients MAY send the `LIST` subcommand to request the list of pinned certificate
fingerprints associated with their account.

If the server cannot process such request, it MUST reply with a standard failure
message with the following format:

```
FAIL CLIENTCERT INTERNAL_ERROR LIST :...
```

Otherwise, the server replies to such request by sending one `LIST` subcommand
per pinned certificate associated with the account the client is logged in.

The server MAY send additional metadata associated with each certificate
fingerprint in the form of additional `key=value` parameters after the
`fingerprint` parameter. Clients MUST ignore unknown keys. The following keys
are defined:

- `name`: The name of the pinned certificate. The value MUST be at least 1 and
  at most 64 bytes.
- `time`: the time of the last successful authentication attempt using the
  pinned certificate. The value MUST follow the format of time values defined by
  the [server-time] extension.
- `ip`: the IP of the last client that successfully authenticated using the
  pinned certificate. The value MUST be either an `IPv6address` or an
  `IPv4address` as defined by [RFC3986].

If the `batch` capability was negotiated, the server MUST reply to a `LIST`
subcommand using a single batch of type `draft/client-cert`. If the list of
pinned certificates is empty, the server MUST reply with an empty batch.

### `CLIENTCERT DELETE` subcommand

The `DELETE` subcommand has the following format for both the client and the
server:

```
CLIENTCERT DELETE [fingerprint]
```

Clients MAY send the `DELETE` subcommand to remove certificates from the list of
pinned certificates.

If the client includes a `fingerprint` parameter, only the pinned certificate
that has this fingerprint may be deleted. Otherwise, only the pinned certificate
that has the fingerprint of the TLS client certificate of the current IRC
connection is deleted.

If the server fails to delete a pinned certificate for any other reason, it MUST
send a standard failure message of the following format:

```
FAIL CLIENTCERT INTERNAL_ERROR DELETE :...
```

If the server successfully processes the `DELETE` subcommand, it MUST reply with
the same `DELETE` subcommand as sent by the client, with the same `fingerprint`
parameter if any.

## Examples

An example of a client that connects with a TLS client certificate, negotiates
the required capabilities, tries to authenticate using its client certificate
then by SASL PLAIN, then creates a pinned certificate but the server changes
the name and finally the client lists all pinned certificates for its account:

```
C: CAP LS 302
S: CAP * LS :batch sasl=PLAIN,EXTERNAL draft/client-cert
C: CAP REQ :batch sasl draft/client-cert
S: CAP * ACK :batch sasl draft/client-cert

C: AUTHENTICATE EXTERNAL
S: :server 904 macron :SASL authentication failed

C: AUTHENTICATE PLAIN
S: AUTHENTICATE +
C: AUTHENTICATE bWFjcm9uAGJnAGJyaWdpdGVqdG0=
S: :server 900 macron macron!macron@macron.fr macron :You are now logged in as macron
S: :server 903 macron :SASL authentication successful

C: CLIENTCERT CREATE :senpai on linux!! 😊
S: CLIENTCERT CREATE

C: CLIENTCERT LIST
S: :server BATCH +asdf draft/client-cert
S: @batch=ok CLIENTCERT LIST F70EC120E1603A58F5979A36221630D92AC75DBAEF8C205B5BB94463E1FBB73BB8E77172BB4CCC4B17FA9F7162AFA80F4E865798DF6CDE7F30E50B0362180E82 name=senpai\son\slinux
S: @batch=ok CLIENTCERT LIST 27776A4894173B93D4E7D4896B9DD617E8132342D1D71CCC4AE341931C30F75EA26316D1C27308D014C034CFDDAFEF879B394439A83BD59C93C6F0CD2B0ADE42 time=2019-01-04T14:34:09.123Z ip=127.0.0.1 name=Goguma\son\sAndroid
S: @batch=ok CLIENTCERT LIST 289C05BC123C3468300FB1205C52F21AB2EF76E77780904C60A82934B967DD719487074CF15D9C9541D3A36F00A1ED2EF4CFF126E7304FADEB1248810AB35020 ip=::1 time=2019-01-04T14:34:16.123Z name=gamja\sat\shttps://macron.gouv.fr/
S: :server BATCH -asdf
```

Then the client reconnects with the same TLS client certificate and deletes the
pinned certificate named "Goguma on Android":

```
C: CAP LS 302
S: CAP * LS :batch sasl=PLAIN,EXTERNAL draft/client-cert
C: CAP REQ :batch sasl draft/client-cert
S: CAP * ACK :batch sasl draft/client-cert

C: AUTHENTICATE EXTERNAL
S: :server 900 macron macron!macron@macron.fr macron :You are now logged in as macron
S: :server 903 macron :SASL authentication successful

C: CLIENTCERT DELETE 27776A4894173B93D4E7D4896B9DD617E8132342D1D71CCC4AE341931C30F75EA26316D1C27308D014C034CFDDAFEF879B394439A83BD59C93C6F0CD2B0ADE42
S: CLIENTCERT DELETE 27776A4894173B93D4E7D4896B9DD617E8132342D1D71CCC4AE341931C30F75EA26316D1C27308D014C034CFDDAFEF879B394439A83BD59C93C6F0CD2B0ADE42
```

[batch]: ../extensions/batch.html
[message-tags]: ../extensions/message-tags.html
[RFC 3986]: https://datatracker.ietf.org/doc/html/rfc3986
[RFC 4648]: https://datatracker.ietf.org/doc/html/rfc4648
[RFC 6234]: https://datatracker.ietf.org/doc/html/rfc6234
[RFC 8446]: https://datatracker.ietf.org/doc/html/rfc8446
[SASL 3.2]: ../extensions/sasl-3.2.html
[server-time]: ../extensions/server-time.html
[standard replies]: ../extensions/standard-replies.html
