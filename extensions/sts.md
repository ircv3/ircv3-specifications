---
title: IRCv3 Strict Transport Security
layout: spec
work-in-progress: true
redirect_from:
  - /specs/core/sts-3.3.html
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

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `sts` capability name. Instead, implementations SHOULD use the `draft/sts` capability name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Description

Strict Transport Security (STS) is a mechanism which allows servers to communicate that complying clients should only connect to them over a secure connection.

The policy is communicated to the client via the STS capability and should be processed by the client at capability negotiation time.

The name of the STS capability is `draft/sts`.

The value of the capability specifies the duration during which the client MUST only connect to the server securely (using TLS, aka. SSL), and the port number to use for secure connections.

Clients MUST use sufficient certificate verification to ensure the connection is secure without any errors. This might involve validation against a system root certificate bundle or a user-specified trust root.

In compliant server software, this capability MUST be made available as an optional configuration setting. Server administrators might have valid reasons not to enable it.

## Details

When enabled, the capability has a REQUIRED value: a comma (`,`) separated list of tokens. Each token consists of a key which might have a value attached. If there is a value attached, the value is separated from the key by a `=`. That is, `<key>[=<value>][,<key2>[=<value2>][,<keyN>[=<valueN>]]]`. Keys specified in this document MUST only occur at most once.

Clients MUST ignore every token with a key that they don't understand.

An STS capability advertised on a secure connection with at least a `duration` key present expresses an **STS persistence policy** for the server at the requested hostname.

An STS capability advertised on an insecure connection with at least a `port` key present expresses an **STS upgrade policy** for the server at the requested hostname.

See [capability negotiation 3.2](capability-negotiation-3.2.html) for more information about capabilities with values.

### Mechanism

When a client sees an STS upgrade policy over a non-secure connection, it MUST first establish a secure connection (see the `port` key) and confirm that the STS persistence policy is present.

Once a client has connected securely, and it has verified that an STS persistence policy is in place, it then MUST only use a secure transport to connect to the server at the requested hostname in future, until the policy expires (see the `duration` key). Once an STS persistence policy has been verified, clients MUST refuse to connect if a secure connection cannot be established to the server for any reason during the lifetime of the policy.

If the secure connection succeeds but the STS persistence policy is not present, the client SHOULD continue using the secure connection for that session. This allows servers to upgrade client connections without committing to a more permanent STS policy.

Clients MUST NOT request this capability (`CAP REQ`). Servers MAY reply with a `CAP NAK` message if a client requests this capability.

Servers MUST NOT send `CAP DEL` to disable this capability, and clients MUST ignore any attempts to do so. The mechanism for disabling an STS persistence policy is described in the `duration` key section.

### The `duration` key

This key is used on secure connections to indicate how long clients MUST continue to use secure connections when connecting to the server at the requested hostname. The value of this key MUST be given as a single integer which represents the number of seconds until the persistence policy expires.

To enforce an STS persistence policy, servers MUST send this key to securely connected clients. Servers MAY send this key to all clients, but non-securely connected clients MUST ignore it.

Clients MUST reset the STS policy expiry time for a requested hostname every time a valid persistence policy with a `duration` key is received from a server.

A duration value of 0 indicates a disabled STS persistence policy with an immediate expiry. This value MAY be used by servers to remove a previously set policy.

When an STS persistence policy expires, clients MAY continue to connect securely to the server at the requested hostname, but they are no longer required to pre-emptively upgrade non-secure connections. Clients MUST still respect any STS upgrade policies encountered when a persistence policy is expired or disabled.

### The `port` key

This key indicates the port number for making a secure connection. This key's value MUST be a single port number.

If the client is not already connected securely to the server at the requested hostname, it MUST close the insecure connection and reconnect securely on the stated port.

To enforce an STS upgrade policy, servers MUST send this key to non-securely connected clients. Servers MAY send this key to securely connected clients, but it will be ignored.

### The `preload` key

This OPTIONAL key, if present, indicates that the server agrees to be included in STS preload lists. If it has a value, the value MUST be ignored.

Clients not looking to confirm whether the server agrees to be included in STS preload lists MAY ignore the presence of this key.

If a client receives a `preload` key on a non-secure connection, it is invalid and MUST be ignored.

See the [Pre-loaded STS policies section](#pre-loaded-sts-policies) for more information on preload lists.

### Server Name Indication

Before advertising an STS policy, servers SHOULD verify whether the hostname provided by clients, for example, via TLS Server Name Indication (SNI), has been whitelisted by administrators in the server configuration.

If no hostname has been provided for the connection, an STS policy SHOULD NOT be advertised.

This allows server administrators to retain control over which hostnames are STS-enabled in case the server is accessible on multiple hostnames. It is possible that a server uses a wildcard certificate or a certificate with Subject Alternative Names but its administrators only wish to advertise STS on a subset of its hostnames.

For example a server presents a wildcard certificate for `*.example.net`. `irc.example.net`, `example.net`, `www.example.net` and `test.example.net` all point to the IP address of the server. The administrators may not wish to have STS enabled for `test.example.net` or `www.example.net`.

IRCd software SHOULD allow for each part of the STS policy to be configured per hostname. This allows server administrators to, for example enable STS persistence on all hostnames, but only enable a preload policy for a subset.

### Rescheduling expiry on disconnect

IRC connections can be long-lived. Connections that last for more than a month are not uncommon. When a client activates an STS persistence policy for a hostname on a long-lived connection, the expiry time might be reached by the time the connection closes. However, the server might still have an STS policy in place.

To avoid an early STS policy expiry, clients MUST reschedule the expiry time when closing connections. The new expiry time is calculated by adding the policy duration as last advertised by the server to the time the connection is closed.

## General Security considerations

This section is non-normative.

### STS policy stripping

It's possible for an attacker to remove the STS `port` value from an initial connection established via an insecure connection, before the policy has been cached by the client. This represents a bootstrap MITM (man-in-the-middle) vulnerability.

Clients might choose to mitigate this risk by implementing features such as user-declared and pre-loaded STS policies, as described in the "Client implementation considerations" section.

### Policy expiry

It is possible that a client successfully receives an STS policy from a server but later on attackers begin to block all secure connection attempts from the client to the server until the policy expires. At that time, the client might revert back to a non-secure connection. Servers should advertise a long enough duration which makes this scenario less likely to happen.

Servers should choose an appropriate `duration` value with reference to the "Server implementation considerations" section to avoid inadvertent expiry issues.

### Denial of Service

STS could result in a Denial of Service (DoS) on IRC servers in a number of ways. Some non-exhaustive examples include:

* An attacker could inject an STS policy into an insecure connection that causes clients to reconnect on a secure port under the attacker's control. If this secure connection succeeds, an unwanted policy could be set for the host and persist in clients even after an administrator has regained control of their server. This can be mitigated in clients by allowing for STS policy rejection as described in the "Client implementation considerations" section.

* A 3rd party host with DNS pointing to an STS-enabled host, where the 3rd party isn't listed on the server's certificate. This configuration would fail certificate validation even without STS, but users might be relying on it for non-secure access.

* An attacker could trick a user into declaring a manual STS policy in their client.

* A server administrator might configure an STS policy for a server whose secure capabilities are later disabled. For example, their host certificate is allowed to expire without being renewed, or is deemed insecure by newly exposed weak cipher suites. Care must be taken when choosing policy expiry times, as discussed in "Server implementation considerations", in particular when hosts are included in client pre-load policy lists. The ability to set a `duration=0` value at any time to revoke an STS policy is also a useful protection against such errors.

These issues are not vulnerabilities with STS itself, but rather are compounding issues for configuration errors, or issues involving vulnerable systems exploited by other means.

## Client implementation considerations

This section is non-normative.

For increased user protection and more advanced management of cached STS policies, clients might consider implementing features such as the following:

### Rescheduling while connected

A client might choose to periodically reschedule policy expiry while connected as well, to protect against sudden power failures, for example.

An appropriate period for rescheduling expiry while connected might depend on the policy's duration. For instance, a longer policy duration might warrant less frequent rescheduling.

### No immediate user recourse

If a client fails to establish a secure connection for any reason to a hostname with an active STS policy, there should be no immediate user recourse to bypass the failure. For example the user should not be presented with a dialog that allows them to proceed with the connection. A failure to establish a secure connection should be treated similarly to a server error that the user is unable to do anything about except retrying at a later time.

This is to ensure that it is not possible to fool users into allowing a man-in-the-middle attack.

### User-declared STS policy

Clients might consider allowing users to explicitly define an STS policy for a given host, before any interaction with the host. This could help prevent a bootstrap MITM vulnerability as discussed in General security considerations.

### Pre-loaded STS policies

As further protection against bootstrap MITM vulnerabilities, clients might choose to include a pre-loaded list of known hosts with STS policies. Such lists should only include hosts with valid certificates and STS policies, and their [preload policy](#the-preload-key) should be in place. This allows IRC network administrators to opt-in to inclusion in preload lists.

Clients should consider how their release upgrade cycle compares to server policy expiry times when compiling pre-loaded lists. Implementations should be able to provide updates to user installed pre-load lists within an appropriate time frame of host policies expiring.

### STS policy deletion or rejection

Clients should consider allowing users or administrators to delete cached STS policies on a per-host basis, in case a server's policy is accidentally or maliciously injected on a secure connection.

Clients might additionally provide the ability to reject STS policies on a per-host basis as an additional mitigation.

These features should be made available very carefully in both the user interface and security senses. Deleting or rejecting a cache entry for a known STS host should be a very deliberate and well-considered act -- it shouldn't be something that users get used to doing as a matter of course: e.g., just "clicking through" in order to get work done. In other words, these features should not violate the "no immediate user recourse" section.

## Server implementation considerations

This section is non-normative.

### Policy expiry time

Network administrators should consider whether their policy duration represents a constant value into the future, or a fixed expiry time.

Constant values into the future can be achieved by a configured number of seconds being sent in the `duration` key on each connection attempt.

Fixed expiry times will involve a dynamic `duration` value being calculated on each connection attempt.

Server implementers should be aware that fixed expiry times might not be precisely guaranteed in the case where clients reschedule policy expiry on disconnect.

Which approach to take will depend on a number of considerations. For example, a server might wish their STS Policy to expire at the same time as their hostname certificate. Alternatively, a server might wish their STS policy to last for as long as possible.

Server implementations should consider using a value of `duration=0` in their example configurations. This will require server administrators to deliberately set an expiry according to their specific needs rather than an arbitrary generic value.

### Offering multiple IRC servers at alternate ports on the same hostname

The STS policy is imposed for all connections to a hostname. This means that mixing STS-enabled and non-secure IRC servers, or running multiple STS-enabled IRC servers on the same hostname on different ports will result in some clients only being able to connect to a single IRC server on that host, depending on which IRC server they connected to first.

For example, a single host might run a production IRC server which advertises an STS policy and another, unrelated IRC server on a different port for testing purposes which does not offer secure connections.

In this case, to allow clients to connect to both IRC servers the non-secure IRC server can be offered at a different hostname, for example a subdomain.

## Relationship with other specifications

This section is non-normative.

### Relationship with STARTTLS

STARTTLS is a mechanism for upgrading a connection which has started out as a non-secure connection to a secure connection without reconnecting to a different port. This means a server can offer both non-secure and secure connections on the same port for compatible clients at the cost of more complex implementations in both clients and servers.

In practice, switching protocols in the middle of the stream has proven to be complicated enough that only a small number of clients bothered implementing STARTTLS.

STS expects that servers offer a port that directly services secure connections and it is incompatible with servers that offer secure connections only via STARTTLS on a non-secure port.

### Relationship with the `tls` cap

The `tls` cap is a hint for clients that secure connection support is available via STARTTLS.

STS is an improved solution and should be considered a replacement for multiple reasons.

* STS is not merely a hint but requires conforming clients to switch to a secure connection. This means that a bookmarked server entry in conforming clients is upgraded to use secure connections, even if the bookmark was originally set up to use non-secure connections.

* STS mandates that clients must only attempt secure connections once a policy has been established, reducing the possibility of users being fooled into allowing a man-in-the-middle attack by downgrading their secure connections to non-secure ones. The `tls` cap has no such provision.

* STS is more flexible, allowing server administrators to configure the lifetime of a policy. 

## Examples

### Redirecting to a secure port with an upgrade policy

A server tells a client connecting non-securely to connect securely on port 6697.

    Non-secure Client: CAP LS 302
               Server: CAP * LS :draft/sts=port=6697

After the exchange, the client disconnects and reconnects securely to the same hostname on port 6697.

### Setting an STS persistence policy with a duration

A server tells a secure client to only use secure connections for roughly 6 months.

    Secure Client: CAP LS 302
           Server: CAP * LS :draft/sts=duration=15552000

Until the policy expires:

* The client will continue use the port it is currently connected on connect securely in future.
* It will not make any non-secure connection attempts to the same hostname.

### Ignoring an invalid request

A server tells a non-secure client to use secure connections for roughly 6 months. There is no port advertised.

    Non-secure Client: CAP LS 302
               Server: CAP * LS :draft/sts=duration=15552000

The client ignores this because it has received an STS persistence policy over a non-secure connection and the STS cap doesn't contain an upgrade policy.

### Handling tokens with unknown keys

A server tells a secure client to use secure connections for roughly a year, but the value of the STS capability also contains tokens the client doesn't recognise.

    Secure Client: CAP LS 302
           Server: CAP * LS :draft/sts=unknown,duration=31536000,foo=bar

The client ignores the keys it does not understand and until the policy expires:

* The client will use the port it is currently connected to in the future to connect securely.
* It will not make any non-secure connection attempts to the same host.

### Handling 0 `duration`

A server tells a client that is already connected securely to remove the STS policy immediately.

    Secure Client: CAP LS 302
           Server: CAP * LS :draft/sts=duration=0

If the client has an STS policy stored for the host it clears the policy. Future attempts to connect non-securely will be allowed.

### Rescheduling expiry on disconnect

A client securely connects to a server, which advertises an STS policy.

    Secure Client: CAP LS 302
           Server: CAP * LS :multi-prefix draft/sts=duration=2592000

The client saves the policy and notes that it will expire in 2592000 seconds (roughly one month). It completes registration, then proceeds as usual.

After 48 hours, the client disconnects.

    Secure Client: QUIT :Bye

The policy is still valid, so the client reschedules expiry for 2592000 seconds from the time of disconnection.

### Receiving CAP DEL

A server makes an invalid attempt to remove an sts policy by disabling the CAP.

    Server: CAP * DEL :draft/sts

The client will ignore this attempt and only rely on receiving a duration of 0 to disable STS policies.


### Enabling an STS preload policy

A client securely connects to a server, which advertises an STS policy and opts in to preload lists.

    Secure Client: CAP LS 302
           Server: CAP * LS :draft/sts=duration=2592000,preload
