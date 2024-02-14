---
title: Bearer tokens
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Shivaram Lingamneni"
    period: "2024"
    email: "slingamn@cs.stanford.edu"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `bearer` CAP name. Instead, implementations SHOULD use the `draft/bearer` CAP name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use unprefixed CAP names.

## Introduction

IRC server implementations may wish to defer authentication to various external systems (e.g. single-sign-on systems). Some of these systems are capable of publishing bearer tokens, i.e., opaque tokens that carry authorization and authentication data, and which can be subsequently be validated by the same system or a cooperating system.

[SASL](sasl-3.1.html) is the standard authentication protocol used in IRC; it offers different mechanisms corresponding to different methods of authentication. Although some bearer tokens have associated SASL mechanisms (for example, OAuth2 has [RFC 7628](https://www.rfc-editor.org/rfc/rfc7628.html)), there is no general mechanism associated with the concept of bearer tokens. Moreover, every additional SASL mechanism imposes an implementation cost on server and client developers. It is therefore desirable to offer a specification for transmitting and processing bearer tokens via the most commonly implemented SASL mechanism: `PLAIN` (which is typically used for authenticating via username and password).

## Implementation

Servers implementing this specification MUST implement [capability negotiation at level 302 or higher](capability-negotiation.html), as well as the [sasl capability](sasl-3.1.html) with the `PLAIN` mechanism.

This specification introduces an informational capability `draft/bearer`. Clients MAY request this capability; servers MUST acknowledge it if requested. The capability takes a value, which is a comma-delimited sequence of bearer token types accepted by the server.

A bearer token type is a case-sensitive identifier conforming to the [message tags](message-tags.html) grammar for `<key_name>` tokens. This specification defines two bearer token types, `oauth2` and `jwt`. Additional bearer token types may be defined; they SHOULD either use a vendor prefix, or be registered with IRCv3.

A bearer token is an opaque string of bytes. Bearer tokens MUST NOT contain the NUL byte.

In order to authenticate to a server by means of a bearer token, a client first obtains a bearer token of the desired type, then authenticates via the `PLAIN` mechanism of SASL, as defined by [RFC 4616](https://datatracker.ietf.org/doc/html/rfc4616) and encoded in the IRCv3 client-to-server protocol as defined by the [SASL extension](sasl-3.1.html).

The client prepends the ASCII characters `*bearer*` to the bearer token type to obtain the PLAIN authentication identity (`authcid`); for example, `oauth2` becomes `*bearer*oauth2`. The client then authenticates via PLAIN with this authentication identity and the bearer token itself as the password (`passwd`). The client MUST either omit the optional authorization identity (`authzid`), or use the authcid again as the authzid.

## Bearer token types

The `oauth2` bearer token type is intended to transport OAuth 2.0 bearer tokens, as defined by [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750).

The `jwt` bearer token type is intended to transport JSON Web Tokens (JWT), as defined by [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519).

## Examples

This is an example of successful authentication with the `jwt` bearer token type:

```
C: CAP LS 302
S: :server.test CAP * LS :account-notify account-tag away-notify batch cap-notify chghost draft/bearer=oauth2,jwt draft/chathistory echo-message extended-join extended-monitor invite-notify labeled-response message-tags multi-prefix sasl=PLAIN,EXTERNAL,SCRAM-SHA-256,OAUTHBEARER server-time setname standard-replies userhost-in-names
C: CAP REQ sasl
S: CAP ACK sasl
C: AUTHENTICATE PLAIN
S: AUTHENTICATE +
C: AUTHENTICATE ACpiZWFyZXIqand0AGV5SmhiR2NpT2lKU1V6STFOaUlzSW5SNWNDSTZJa3BYVkNKOS5leUp3Y21WbVpYSnlaV1JmZFhObGNtNWhiV1VpT2lKemJHbHVaMkZ0YmlKOS5jYVBadzJEbDRLWk4tU0VyRDUtV1pCX2xQUHZlSFhhTUNvVUh4TmViYjk0Rzl3M1ZhV0RJUmRuZ1ZVOTlKS3g1bkVfeVJ0cGV3a0hIdlhzUW5OQV9NNjNHQlhHSzdhZlhCOGUta1YzM1FGM3Y5cFhBTE1QNVN6UndNZ29reXhhczBSZ0h1NGU0TDBkN2RuOW9fbmtkWHAzNEdYM1BuMU1Wa1VHQkg2R2RsYk9kREhyczA0cFBRMFFqLU8yVTBBSXBuWnEtWF9HUXM5RUNK
C: AUTHENTICATE bzRUbFBLV1I3SmxxNWw5YlMwZEJub2hlYTRGdXFKcjIzMmplLWRsUlZrYkNhN25ybkZtc0lzZXpzZ0EzSmJfajladV9pdjQ2MHRfZDJlYXl0YlZwOVAtRE9WZnpVZmtCc0tzLTgxVVJRRW5Ualc2dXQ0NDVBSnoycHhqWDkyWDBHZG1PUnBBa1E=
S: :server.test 900 * * slingamn :You are now logged in as slingamn
S: :server.test 903 * :Authentication successful
C: NICK slingamn
C: USER u s e r
C: CAP END
S: :server.test 001 slingamn :Welcome to the IRC Network slingamn
```

## Implementation considerations

This section is non-normative.

Bearer token types, as defined in this specification, are not intended to transmit complete information about how bearer tokens are to be generated or validated. In general, it will be necessary to configure both client and server out of band so that they agree on these issues. However, this specification leaves open the possibility that further capabilities could be defined, allowing the client to discover a token issuance mechanism.

Extraction of the IRC account name (or other applicable IRC-level authentication data) from the token is left implementation-defined, as part of the validation process. However, the `oauth2` token type is intended for validation via an [OAuth 2.0 introspection endpoint](https://www.oauth.com/oauth2-servers/token-introspection-endpoint/); in this case, the server will typically validate the `active` member of the introspection response, then use the `username` member as the client's IRC account name. For the `jwt` token type, possibilities include using the local-part of a `sub` claim formatted as an e-mail address, using the `preferred_username` claim as defined by the [IANA JWT claims registry](https://www.iana.org/assignments/jwt/jwt.xhtml), or using a custom claim.

Since the base64-encoded size of a PLAIN response containing a bearer token is likely to exceed 400 bytes, clients should implement the ability to emit multi-line `AUTHENTICATE` responses, as defined in the [SASL](sasl-3.1.html) specification. Servers should accept multi-line `AUTHENTICATE` responses long enough to accommodate any valid token that can arise in practice.
