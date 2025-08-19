---
title: IRCV3BEARER SASL mechanism
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

## Introduction

IRC server implementations may wish to defer authentication to various external systems (e.g. single-sign-on systems). Some of these systems are capable of publishing bearer tokens, i.e., opaque tokens that carry authorization and authentication data, and which can be subsequently be validated by the same system or a cooperating system.

[SASL](sasl-3.1.html) is the standard authentication protocol used in IRC; it offers different mechanisms corresponding to different methods of authentication. Although some bearer tokens have associated SASL mechanisms (for example, OAuth2 has [RFC 7628](https://www.rfc-editor.org/rfc/rfc7628.html)), there is no general mechanism associated with the concept of bearer tokens. This specification defines a new mechanism `IRCV3BEARER` for processing bearer tokens in the context of IRCv3.

## Implementation

Servers implementing this specification MUST implement [capability negotiation at level 302 or higher](capability-negotiation.html), as well as the [sasl capability](sasl-3.1.html).

A bearer token type is a case-sensitive identifier conforming to the [message tags](message-tags.html) grammar for `<key_name>` tokens. This specification defines two bearer token types, `oauth2` and `jwt`. Additional bearer token types may be defined; they SHOULD either use a vendor prefix, or be registered with IRCv3.

A bearer token is an opaque string of bytes. Bearer tokens MUST NOT contain the NUL byte.

In order to authenticate to a server by means of a bearer token, a client first obtains a bearer token of the desired type, then initiates a SASL conversation with the server using the mechanism `IRCV3BEARER`. This mechanism consists of a single message from the client to the server, having the form:

    <message>   ::= [authzid] NUL <token_type> NUL <token>

where the optional authzid (authorization identity) is as specified by [RFC 4616](https://datatracker.ietf.org/doc/html/rfc4616).

## Bearer token types

The `oauth2` bearer token type is intended to transport OAuth 2.0 bearer tokens, as defined by [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750).

The `jwt` bearer token type is intended to transport JSON Web Tokens (JWT), as defined by [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519).

## Examples

This is an example of successful authentication with the `jwt` bearer token type:

```
C: CAP LS 302
S: :server.test CAP * LS :account-notify account-tag away-notify batch cap-notify chghost jwt draft/chathistory echo-message extended-join extended-monitor invite-notify labeled-response message-tags multi-prefix sasl=PLAIN,EXTERNAL,SCRAM-SHA-256,OAUTHBEARER,IRCV3BEARER server-time setname standard-replies userhost-in-names
C: CAP REQ sasl
S: CAP ACK sasl
C: AUTHENTICATE IRCV3BEARER
S: AUTHENTICATE +
C: AUTHENTICATE AGp3dABleUpoYkdjaU9pSlNVekkxTmlJc0luUjVjQ0k2SWtwWFZDSjkuZXlKd2NtVm1aWEp5WldSZmRYTmxjbTVoYldVaU9pSnpiR2x1WjJGdGJpSjkuY2FQWncyRGw0S1pOLVNFckQ1LVdaQl9sUFB2ZUhYYU1Db1VIeE5lYmI5NEc5dzNWYVdESVJkbmdWVTk5Skt4NW5FX3lSdHBld2tISHZYc1FuTkFfTTYzR0JYR0s3YWZYQjhlLWtWMzNRRjN2OXBYQUxNUDVTelJ3TWdva3l4YXMwUmdIdTRlNEwwZDdkbjlvX25rZFhwMzRHWDNQbjFNVmtVR0JINkdkbGJPZERIcnMwNHBQUTBRai1PMlUwQUlwblpxLVhfR1FzOUVDSm80VGxQS1dS
C: AUTHENTICATE N0pscTVsOWJTMGRCbm9oZWE0RnVxSnIyMzJqZS1kbFJWa2JDYTducm5GbXNJc2V6c2dBM0piX2o5WnVfaXY0NjB0X2QyZWF5dGJWcDlQLURPVmZ6VWZrQnNLcy04MVVSUUVuVGpXNnV0NDQ1QUp6MnB4alg5MlgwR2RtT1JwQWtR
S: :server.test 900 * * slingamn :You are now logged in as slingamn
S: :server.test 903 * :Authentication successful
C: NICK slingamn
C: USER u s e r
C: CAP END
S: :server.test 001 slingamn :Welcome to the IRC Network slingamn
```

## Implementation considerations

This section is non-normative.

This specification does not specify how token types are to be discovered, or how tokens are to be issued or validated. In general, it will be necessary to configure both client and server out of band so that they agree on these issues. However, the possibility is open for a future specification to allow client discovery of available token types and issuance mechanisms.

Extraction of the IRC account name (or other applicable IRC-level authentication data) from the token is left implementation-defined, as part of the validation process. However, the `oauth2` token type is intended for validation via an [OAuth 2.0 introspection endpoint](https://datatracker.ietf.org/doc/html/rfc7662#section-2); in this case, the server will typically validate the `active` member of the introspection response, then use the `username` member as the client's IRC account name. For the `jwt` token type, possibilities include using the local-part of a `sub` claim formatted as an e-mail address, using the `preferred_username` claim as defined by the [IANA JWT claims registry](https://www.iana.org/assignments/jwt/jwt.xhtml), or using a custom claim.

Since the base64-encoded size of a PLAIN response containing a bearer token is likely to exceed 400 bytes, clients should implement the ability to emit multi-line `AUTHENTICATE` responses, as defined in the [SASL](sasl-3.1.html) specification. Servers should accept multi-line `AUTHENTICATE` responses long enough to accommodate any valid token that can arise in practice.
