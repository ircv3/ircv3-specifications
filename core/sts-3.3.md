---
title: IRCv3.3 Strict Transport Security
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Attila Molnar"
    period: "2015-2016"
    email: "attilamolnar@hush.com"
  -
    name: "James Wheare"
    period: "2016"
    email: "james@irccloud.com"
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
is secure without any errors. This may involve validation against a system root
certificate bundle or a user-specified trust root.

In compliant server software, this capability MUST be made available as an optional
configuration setting. Server administrators may have valid reasons not to enable it.

## Relationship with other specifications

### Relationship with STARTTLS

STARTTLS is a mechanism for upgrading a connection which has started out as a
non-secure connection to a secure connection without reconnecting to a
different port. This means a server can offer both non-secure and secure
connections on the same port for compatible clients at the cost of more
complex implementations in both clients and servers.

In practice, switching protocols in the middle of the stream has proven to be
complicated enough that only a small number of clients bothered implementing
STARTTLS.

This specification expects that servers offer a port that directly services
secure connections and it is incompatible with servers that offer secure
connections only via STARTTLS on a non-secure port.

### Relationship with the `tls` cap

The `tls` cap is a hint for clients that secure connection support is
available via STARTTLS.

This specification is an improved solution to that and should be considered a
replacement for it for multiple reasons.

* This specification is not merely a hint but requires conforming clients to
switch to a secure connection. This means that a bookmarked server entry in
conforming clients is upgraded to use secure connections, even if the
bookmark was originally set up to use non-secure connections.
* This specification mandates that clients must only attempt secure connections
to a server which has a policy, reducing the possibility of users being fooled
into allowing a man-in-the-middle attack by downgrading their secure
connections to non-secure ones. The `tls` cap has no such provision.
* This specification is more flexible, allowing server administrators to
configure the lifetime of the policy. 

## Details

When enabled, the capability has a REQUIRED value: a comma (`,`) separated
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

### Handling disconnection

IRC connections may be long-lived. Connections lasting for more than a month
are not uncommon. When a client has successfully learned the STS policy for a
server but it does not reconnect for a long period of time it may think
the policy expired, even if in fact the same policy is still advertised by the
server.

The client MUST reschedule the expiration on disconnection.
The new expiration is calculated from the current policy as last advertised by
the server and the current time.

## Client implementation considerations

This section is non-normative.

For increased user protection and more advanced management of cached STS policies,
clients should consider implementing features such as the following:

### Rescheduling while connected

The client may choose to periodically reschedule the expiration of the policy
while connected as well, to protect against sudden power outages, for example.

An appropriate period for rescheduling expiration while connected might
depend on the length of the policy expiration. For instance, longer expiry
times might warrant less frequent rescheduling.

### No immediate user recourse

If a client fails to establish a secure connection for any reason to a server with which
an STS policy has been stored, there should be no immediate user recourse. For example
the user should not be presented with a dialog that allows them to proceed with
the connection. A failure to establish a secure connection should be treated similarly
to a server error that the user is unable to do anything about except retrying at a
later time.

This is to ensure that it is not possible to fool users into allowing a
man-in-the-middle attack.

### User-declared STS policy

Clients may allow users to explicitly define an STS policy for a given host, before any
interaction with the host. This could help prevent a "Bootstrap MITM vulnerability" as
discussed in General security considerations.

### Pre-loaded STS policies

As further protection against bootstrap MITM vulnerabilities, clients may choose to
include a pre-loaded list of known hosts with STS policies. Such lists should be
compiled on an opt-in basis, by request of IRC network administrators. Hosts should be
verified to be correctly advertising an STS policy before inclusion.

Clients should consider how their release upgrade cycle compares to server policy expiry
times when compiling pre-loaded lists. Implementations should be able to provide updates
to user installed pre-load lists within an appropriate time frame of host policies expiring.

### STS policy deletion or rejection

Clients should allow users to delete cached STS policies on a per-host
basis, in case a server's policy is accidentally or maliciously injected on
a secure connection.

Clients may additionally provide the ability to reject STS policies on a
per-host basis as an additional mitigation.

These features should be made available very carefully in both the user interface
and security senses. Deleting or rejecting a cache entry for a known STS host should
be a very deliberate and well-considered act -- it shouldn't be something that users
get used to doing as a matter of course: e.g., just "clicking through" in order
to get work done. In other words, these features should not violate the "no immediate
user recourse" section.

## Server implementation considerations

This section is non-normative.

### Policy expiry time

IRCd configurations and network administrators should consider whether expiry times
represent a constant value into the future, or a fixed expiry time.

Constant values into the future can be achieved by a configured number of seconds being
sent in the `duration` key on each connection attempt.

Servers implementors should be aware that fixed expiry times may not be precisely
guaranteed in the case where clients reschedule policy expiry on disconnect.

Fixed expiry times will involve a dynamic `duration` value being calculated on each
connection attempt.

Which approach to take will depend on a number of considerations. For example, a server
may wish their STS Policy to expire at the same time as their domain certificate.
Alternatively, a server may wish their STS policy to last for as long as possible.

Server implementations should consider using a value of `duration=0` in their example
configurations. This will require server administrators to deliberately set an expiry
according to their specific needs rather than an arbitrary generic value.

### Offering multiple IRC servers at alternate ports on the same domain

The STS policy is imposed for an entire domain name. This means that mixing
STS-utilizing and non-secure IRC servers on the same domain name or running
multiple STS-utilizing IRC servers on the same domain name may result in some
clients only being able to connect to a single IRC server on that domain name,
depending on which IRC server they connected to first.

For example, a single server may run a production IRC server which advertises
an STS policy and another, unrelated IRC server on a different port for
testing purposes which does not offer secure connections.

In this case, to allow clients to connect to both IRC servers the non-secure IRC
server may be offered at a different domain name, for example a subdomain.

## General Security considerations

### STS policy stripping

It's possible for an attacker to strip the STS `port` value from an initial connection
established via an insecure connection, before the policy has been cached by the client.
This represents a Bootstrap MITM (man-in-the-middle) vulnerability.

Clients may choose to mitigate this risk by implementing features such as user-declared
and pre-loaded STS policies, as described in the "Client implementation considerations"
section.

### Policy expiry

It is possible that a client successfully receives the STS policy from a server
but later on attackers begin to block all secure connection attempts from the
client to the server until the policy expires. At that time, the client will
revert back to a non-secure connection. Servers SHOULD advertise a long enough
duration which makes this scenario less likely to happen.

Servers should choose an appropriate `duration` value according with reference to
the "Server implementation considerations" section to avoid inadvertent expiry issues.

### Denial of Service

STS could result in a Denial of Service (DoS) on IRC servers in a number of ways.
Some non-exhaustive examples include:

* An attacker could inject an STS policy into an insecure connection that causes clients
to reconnect on a secure port under the attacker's control. If this secure connection
suceeds, an unwanted policy will be set for the host and persist in clients even after
an administrator has regained control of their server. This can be mitigated in clients
by allowing for STS policy rejection as described in the "Client implementation
considerations" section.

* A 3rd party host that points to an STS-enabled host via DNS alias, where the 3rd party
isn't listed on the server's certificate. This configuration would fail certificate validation
even without STS, but users may be relying on it for unsecure access.

* An attacker could trick a user into declaring a manual STS policy in their client.

* A server administrator may configure an STS policy for a server whose secure capabilities
are later disabled. For example, their host certificate is allowed to expire without being
renewed, or is deemed insecure by newly exposed weak hash algorithms. Care must be taken when
choosing policy expiry times, as discussed in "Server implementation considerations", in
particular when hosts are included in client pre-load policy lists. The ability to set a
`duration=0` value at any time to revoke an STS policy is also a useful protection against
such errors.

These issues are not vulnerabilities with STS as such, but rather compound configuration
errors, or issues involving vulnerable systems exploited by other means.

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
    Server: CAP * LS :sts=duration=15552000

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

### Rescheduling on disconnection

A client securely connects to a server, which advertises an STS policy.

    Client: CAP LS 302
    Server: CAP * LS :multi-prefix sts=duration=2592000

The client saves the policy and notes that it will become expired in 2592000 seconds
(roughly one month). It completes registration, then proceeds as usual.

After 48 hours, the client disconnects.

    Client: QUIT :Bye

According to the client's last information at the time of disconnection the
policy is still valid, so it reschedules the expiration to occur in 2592000
seconds from the time of disconnection.
