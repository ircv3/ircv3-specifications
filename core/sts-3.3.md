---
title: IRCv3.3 Strict Transport Security
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Attila Molnar"
    period: "2015"
    email: "attilamolnar@hush.com"
---

## Description

Strict Transport Security (STS) is a mechanism which allows servers to
communicate that complying clients should only connect to them over a secure
connection.

The policy is communicated to the client via the `sts` capability and should
be processed by the client at capability negotiation time.

The value of the capability specifies the duration during which the client
MUST only connect to the server securely (using TLS, aka. SSL), and the port
number to use for it.

Clients MUST use sufficient certificate verification to ensure the connection
is secure without any errors.

## Details

The capability value, which MUST always be present, is a comma (`,`) separated
list of tokens. Each token is a key which may have a value attached. If there
is a value attached, the value is separated from the key by a `=`. That is,
`<key>[=<value>][,<key2>[=<value2>][,<keyN>[=<valueN>]]]`. Keys specified in
this document MUST only occur at most once.

Clients MUST ignore every token with a key that they don't understand.

An `sts` capability having at least a `duration` key expresses an STS policy.

See [capability negotiation 3.2](capability-negotiation-3.2.html) for more
information about capabilities with values.

### Mechanism

When the client sees the `sts` capability advertised over a non-secure
connection, it MUST first establish a secure connection and confirm that the
`sts` capability is still present.

Once the client has confirmed that the `sts` capability is indeed offered over
a secure connection, it then MUST only attempt secure connections to the
server from now on until the policy expires (see the `duration` key).
It MUST refuse to connect if a secure connection cannot be established with the
server for any reason during the lifetime of the policy.

However, if the client fails to connect securely for any reason, the connection
attempt SHOULD be considered a failure, similar to a network error.
The client SHOULD retry the secure connection next time it receives the `sts`
cap with the appropriate keys over a non-secure connection.

If the secure connection succeeds but the STS policy is not present, the client
SHOULD continue using the secure connection for that session.

Clients MUST NOT request this capability (`CAP REQ`).
Servers MUST reply with an appropriate `CAP NAK` message if a client requests
this capability.

### The `duration` key

This key indicates how long from receiving the advertisement clients MUST use a
secure connection when connecting to this server. This MUST be specified in
seconds as the value of this key and as a single integer without a prefix or
suffix.

Servers MUST send this key to securely connected clients.
Servers MAY send this key to non-securely connected clients, but it will be
ignored.

If the client receives the `duration` key on a non-secure connection, it is
invalid and MUST be ignored.

Clients MUST reset the duration "time to live" every time they receive a valid
policy advertisement having a `duration` key from the server.

A duration of 0 means the policy expires immediately. This method can be used
by servers to remove a previously set policy.

### The `port` key

This key indicates the port number that clients can use when connecting over
TLS. This key's value MUST be a single port number.

If the client is not already connected securely, it MUST close the insecure
connection and reconnect securely on the stated port.

Servers MUST send this key to non-securely connected clients.
Servers MAY send this key to securely connected clients, but it will be
ignored.

## No user recourse

User recourse should not be possible if the client fails to establish a
secure connection for any reason with a server which the client has a STS
policy for. For example the user should not be presented with a dialog that
allows them to proceed with the connection. A failure to establish a secure
connection should be treated similarly to a server error that the user is
unable to do anything about except retrying at a later time.

This is to ensure that it is not possible to fool users into allowing a
man-in-the-middle attack.

## Security considerations

### Acceptance of STS policy

STS policies SHOULD ONLY be accepted for connections where the TLS certificate
chain can be validated entirely, to either a user-specified root or the system
root certificate bundle.

### Visual feedback of STS upgraded connections

Clients SHOULD NOT display visual feedback indicating that an STS-upgraded TLS
connection has any enhanced security over a plaintext connection unless a user
explicitly trusts the certificate.

### STS policy injection

It is possible for attackers to inject the STS policy into a plaintext
connection. This causes the client to reconnect over TLS and either fail,
or connect successfully over TLS but discover that the `sts` cap is not
present.

Additionally, it is possible for an attacker to inject an STS policy into
a proxied TLS connection. This causes the client to continue following the
STS policy once the proxy is removed, which may cause a denial of service.
Clients SHOULD provide the ability to flush any relevant STS cache entries
for the server to allow for re-learning the STS policy for that server, if
any. Clients MAY additionally provide the ability to reject STS policies
from servers the user knows to not provide an STS policy as an additional
mitigation.

### Policy expiration

It is possible that a client successfully receives the STS policy from a server
but later on attackers begin to block all secure connection attempts from the
client to the server until the policy expires. At that time, the client will
revert back to a non-secure connection. Servers SHOULD advertise a long enough
duration which makes this scenario less likely to happen.

## Examples

### Redirection to a secure port

Server tells a client connecting non-securely to connect securely on port 6697.

    Client: CAP LS 302
    Server: CAP * LS :sts=port=6697

After the exchange, the client disconnects and reconnects securely to the same
server on port 6697 and proceeds as it normally would.

### Setting STS policy

Server tells a client that is already connected securely that the client must
only use secure connections for roughly 6 months.

    Client: CAP LS 302
    Server: CAP * LS :sts=duration=15552000

Until the policy expires:
* The client will use the port it is currently connected to in the future to
connect securely.
* It will not make any non-secure connection attempts to the same server.

### Ignoring invalid request

Server tells a client that is connected non-securely that the client must
use secure connections for roughly 6 months. There is no port advertised.

    Client: CAP LS 302
    Server: CAP * LS :sts=duration=31536000

The client ignores this because it received the STS policy over a non-secure
connection and the `sts` cap contains no token with key `port`.

### Handling tokens with unknown keys

Server tells a client that is already connected securely that the client must
use secure connections for roughly a year, but the value of the `sts` capability
also contains some tokens whose keys the client does not understand.

    Client: CAP LS 302
    Server: CAP * LS :sts=unknown,duration=31536000,foo=bar

The client ignores the keys it does not understand and until the policy
expires:
* The client will use the port it is currently connected to in the future to
connect securely.
* It will not make any non-secure connection attempts to the same server.

### Handling 0 `duration`

Server tells a client that is already connected securely to remove the STS
policy now.

    Client: CAP LS 302
    Server: CAP * LS :sts=duration=0

If the client has any STS policy stored for the server it clears the policy.
Future connections should use whatever settings (port, secure/non-secure) the
client used before it received the STS policy.

## TODO

* SNI integration, `sub` key: [issue #176](https://github.com/ircv3/ircv3-specifications/issues/176)
* Ensure that clients with long lived connections renew the policy: [issue #178](https://github.com/ircv3/ircv3-specifications/issues/178)
* Advertise hosts the server has a valid certificate for (?)
* Mention STARTTLS
