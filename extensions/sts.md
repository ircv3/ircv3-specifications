---
title: Strict Transport Security
layout: spec
redirect_from:
  - /specs/core/sts-3.3.html
copyrights:
  -
    name: "Attila Molnar"
    period: "2015-2017"
    email: "attilamolnar@hush.com"
  -
    name: "James Wheare"
    period: "2016-2017"
    email: "james@irccloud.com"
---

## Description

Strict Transport Security (STS) is a mechanism which allows servers to advertise a policy that clients should only connect to them over a secure connection.

The policy is communicated to clients via the STS capability and should be processed by the client at capability negotiation time.

The name of the STS capability is `sts`.

The value of the capability specifies the duration during which the client MUST only connect to the server securely (using TLS, aka. SSL), and the port number to use for secure connections.

Clients MUST use sufficient certificate verification to ensure the connection is secure without any errors. This might involve validation against a system root certificate bundle or a user-specified trust root.

In server software, this capability MUST be made available as an optional configuration setting. Server administrators might have valid reasons not to enable it.

## Details

When enabled, the capability has a REQUIRED value: a comma (`,`) (0x2C) separated list of tokens. Each token consists of a key which might have a value attached. If there is a value attached, the value is separated from the key by an equals sign (`=`) (0x3D). That is, `<key>[=<value>][,<key2>[=<value2>][,<keyN>[=<valueN>]]]`. Keys specified in this document MUST only occur at most once.

Clients MUST ignore every token with a key that they don't understand.

An STS policy has several parts:

* An **upgrade policy**, expressed via the [`port` key](#the-port-key). *REQUIRED* on an insecure connection.
* A **persistence policy**, expressed via the [`duration` key](#the-duration-key) *REQUIRED* on a secure connection.
* A **preload policy**, expressed via the [`preload` key](#the-preload-key) *OPTIONAL* on a secure connection.

If any required part is missing from the STS policy, clients MUST continue as if no STS policy was advertised.

Servers MAY advertise all STS policy parts together on both secure and insecure connections. Clients MUST only respect each policy part on the appropriate connection type.

If any required key does not exist, clients MUST silently ignore the policy.

See the [capability negotiation](../core/capability-negotiation.html) specification for more information about capabilities with values.

### Mechanism

When a client sees an STS upgrade policy over an insecure connection, it MUST first establish a secure connection (see the [`port` key](#the-port-key)) and confirm that the STS persistence policy is present.

Once a client has connected securely, and it has verified that an STS persistence policy is in place, it then MUST only use a secure transport to connect to the server at the requested hostname in future, until the policy expires (see the [`duration` key](#the-duration-key)). Once an STS persistence policy has been verified, clients MUST refuse to connect if a secure connection cannot be established to the server for any reason during the lifetime of the policy.

If the secure connection succeeds but an STS persistence policy is not present, the client SHOULD continue using the secure connection for that session. This allows servers to upgrade client connections without committing to a more permanent STS policy.

Clients MUST NOT request this capability with `CAP REQ`. Servers MAY reply with a `CAP NAK` message if a client requests this capability.

Servers MAY communicate changes to their STS persistence policy using the `CAP NEW` command provided by `cap-notify` or [CAP Version `302`](../core/capability-negotiation.html). Clients MUST store or update an STS policy for the hostname of a securely connected server if they receive a new STS capability notification.

Servers MUST NOT send `CAP DEL` to disable this capability, and clients MUST ignore any attempts to do so. The mechanism for disabling an STS persistence policy is described in the `duration` key section.

### The `port` key

This key indicates the port number for making a secure connection. This key's value MUST be a single port number.

If the client is not already connected securely to the server at the requested hostname, it MUST close the insecure connection and reconnect securely on the stated port.

To enforce an STS upgrade policy, servers MUST send this key to insecurely connected clients. Servers MAY send this key to securely connected clients, but it will be ignored.

### The `duration` key

This key is used on secure connections to indicate how long clients MUST continue to use secure connections when connecting to the server at the requested hostname. The value of this key MUST be given as a single integer which represents the number of seconds until the persistence policy expires.

To enforce an STS persistence policy, servers MUST send this key to securely connected clients. Servers MAY send this key to all clients, but insecurely connected clients MUST ignore it.

Clients MUST reset the STS policy expiry time for a requested hostname every time a valid persistence policy with a `duration` key is received on a secure connection.

A `duration=0` value indicates a disabled STS persistence policy with an immediate expiry. This value MAY be used by servers to remove a previously set policy.

When an STS persistence policy expires, clients MAY continue to connect securely to the server at the requested hostname, but they are no longer required to pre-emptively upgrade insecure connections. Clients MUST still respect any STS upgrade policies encountered when a persistence policy is expired or disabled.

### The `preload` key

This OPTIONAL key, if present on a secure connection, indicates that the server agrees to be included in STS preload lists. If it has a value, the value MUST be ignored.

Servers MAY send this key to all clients, but insecurely connected clients MUST ignore it.

Preload list providers MUST only consider hosts for inclusion after validating their connection security and ensuring a valid STS policy with a `preload` key is in place. This allows IRC network administrators to opt-in for inclusion in preload lists.

Servers SHOULD be prepared to offer secure connections for the long term when enabling a preload policy. Timely removal of hostnames from preload lists might not be possible.

Preload list providers SHOULD consider STS persistence policy durations and MAY set minimum duration requirements prior to inclusion. Clients using preload lists SHOULD consider how their release cycle compares to any duration requirements imposed by list providers.

### The `if-host-match` and `port-if-match` keys

Sometimes, third-parties create domain aliases that point towards IRC networks (for example, setting `irc.ircv3.net` as a `CNAME` of `irc.example.com`). Unfortunately, this means that TLS certificate validation does not work on the alias, and the network will not be able to use STS without breaking clients that do connect using the alias (e.g. if a user connects via `irc.ircv3.net`, they will not be able to validate the TLS certificate as the cert is valid for `irc.example.com`).

Networks that require this sort of control over connection upgrades can advertise these two keys. Servers that advertise these two keys MUST NOT advertise the regular `port` key:

- `if-host-match`: List of hostnames and hostname patterns to match against, separated by pipes (`|`). This uses regular IRC 'glob' syntax, where `?` matches any single character and `*` matches zero or more characters.
- `port-if-match`: This is the port to connect to.

If the hostname that the client connected to is matched by the `if-host-match` list, the client MUST follow the regular STS process using the `port-if-match` key as the TLS port to connect to.

Advertising these two keys and not advertising the `port` key means that only clients who understand how to process the `if-host-match` key will follow the STS policy.

Example 1: `irc.ircv3.net` is an alias of the canonical server `irc.example.com`. Alice connects to `irc.ircv3.net` and sees the capability: `"sts=duration=1,if-host-match=*.example.com,port-if-match=6697"`. Because Alice connected to `irc.ircv3.net` and does not see a pattern matching that, Alice does not perform an STS upgrade.

Example 2: Alice connects to `irc.example.com` and sees the capability: `"sts=duration=1,if-host-match=*.example.net|*.example.com,port-if-match=6697"`. The pattern `*.example.com` matches `irc.example.com`, so Alice connects to port `6697` and follows the STS process as usual.

### Server Name Indication

Before advertising an STS persistence policy over a secure connection, servers SHOULD verify whether the hostname provided by clients, for example, via TLS Server Name Indication (SNI), has been whitelisted by administrators in the server configuration.

If no hostname has been provided for the connection, an STS persistence policy SHOULD NOT be advertised.

This allows server administrators to retain control over which hostnames are STS-enabled in case the server is accessible on multiple hostnames. It is possible that a server uses a wildcard certificate or a certificate with Subject Alternative Names but its administrators only wish to advertise STS on a subset of its hostnames.

Take for example a server presenting a wildcard certificate for `*.example.net`. The hostnames `irc.example.net`, `example.net`, `www.example.net` and `test.example.net` all point to the same IP address. The server administrators may only wish to have STS enabled for `irc.example.net`, but no other hostname.

IRCd software SHOULD allow for each part of the STS policy to be configured per hostname. This allows server administrators to, for example, enable STS persistence on all hostnames, but only enable a preload policy for a subset of them.

SNI hostname verification is not available on insecure connections, so it might not be possible to configure a variable upgrade policy for multiple hostnames that share an IP address.

### Rescheduling expiry on disconnect

IRC connections can be long-lived. Connections that last for more than a month are not uncommon. When a client activates an STS persistence policy for a hostname on a long-lived connection, the expiry time might be reached by the time the connection closes. However, the server might still have an STS policy in place.

To avoid an early STS policy expiry, clients MUST reschedule the expiry time when closing connections. The new expiry time is calculated by adding the policy duration as last advertised by the server to the time the connection is closed.

## General Security considerations

This section is non-normative.

### STS policy stripping

It's possible for an attacker to remove the STS `port` value from an initial connection established via an insecure connection, before the policy has been cached by the client. This represents a bootstrap MITM (man-in-the-middle) vulnerability.

Clients might choose to mitigate this risk by implementing features such as [user-declared](#user-declared-sts-policy) and [preloaded](#the-preload-key) STS policies.

### Denial of Service

STS could result in a Denial of Service (DoS) on IRC servers in a number of ways. A non-exhaustive list of examples might include:

* An attacker could inject an STS policy into an insecure connection that causes clients to reconnect on a secure port under the attacker's control. If this secure connection succeeds, an unwanted policy could be set for the host and persist in clients even after an administrator has regained control of their server.

* A 3rd party sets up a hostname with DNS pointing to a server they don't control, and the hostname isn't covered by the server's certificate. This configuration already fails certificate validation, but users might be relying on it for insecure access. If the server administrators enable STS on the server, users would no longer be able to connect with that hostname.

* An attacker could trick a user into declaring a manual STS policy in their client for an insecure hostname.

* A server administrator might configure an STS policy for a server whose secure capabilities are later disabled. For example, their host certificate is allowed to expire without being renewed, or is deemed insecure by newly exposed weak cipher suites.

These issues are not vulnerabilities with STS itself, but rather are compounding issues for configuration errors, or issues involving vulnerable systems exploited by other means.

Clients and servers can mitigate many of these issues by adopting mechanisms such as preload lists described in the following implementation consideration sections 

## Client implementation considerations

This section is non-normative.

For increased user protection and more advanced management of cached STS policies, clients might consider implementing features such as the following:

### Rescheduling while connected

In addition to [rescheduling expiry on disconnect](#rescheduling-expiry-on-disconnect), clients might choose to periodically reschedule policy expiry while connected as well. This could provide extra protection in the case of a sudden power failure, for example.

An appropriate period for rescheduling expiry while connected might depend on the policy's duration. For instance, a longer policy duration might warrant less frequent rescheduling.

### No immediate user recourse

If a client fails to establish a secure connection to a hostname with an active STS policy, there should be no immediate user recourse to bypass the failure. Clients should treat a failure to establish a secure connection to an STS-enabled host as a critical error, and shouldn't offer a prominent way to retry it insecurely. For example, the user should not be presented with a dialog that allows them to proceed with the connection with a click. Users should instead be required to retry the connection manually at a later time.

This advice is intended to avoid teaching users that strict security errors can be ignored, and to offer an extra layer of protection from man-in-the-middle attacks.

### User-declared STS policy

Clients might consider allowing users to explicitly define an STS policy for a given host, before any interaction with the host. This could help prevent a bootstrap MITM vulnerability as discussed in the [STS policy stripping](#sts-policy-stripping) section.

### STS policy deletion or rejection

Clients might consider allowing users or administrators to reject or remove cached STS policies on a per-host basis, in case a server's policy is accidentally or maliciously injected on a secure connection.

Such a feature should be made available very carefully from both user interface and security standpoints. Deleting or rejecting a cache entry for a known STS host should be a very deliberate and well-considered act -- it shouldn't be something that users get used to doing as a matter of course: e.g., just "clicking through" in order to get work done. In other words, these features should not violate the [no immediate user recourse](#no-immediate-user-recourse) section.

## Server implementation considerations

This section is non-normative.

### Policy expiry time

Network administrators should consider whether their policy duration represents a constant value into the future, or a fixed expiry time.

Constant values into the future can be achieved by a configured number of seconds being sent in the `duration` key on each connection attempt.

Fixed expiry times will involve a dynamic `duration` value being calculated on each connection attempt.

Server implementers should be aware that fixed expiry times might not be precisely guaranteed in the case where clients reschedule policy expiry on disconnect, or periodically while connected.

Which approach to take will depend on a number of considerations. For example, a server might wish their STS Policy to expire at the same time as their hostname certificate. Alternatively, a server might wish their STS policy to last for as long as possible.

Server administrators should be aware of the ability to set a `duration=0` value at any time to revoke an STS policy if secure connections are no longer enabled for their servers.

Server implementations should consider using a default value of `duration=0` in their example configurations. This will require server administrators to deliberately choose an expiry according to their specific needs rather than (perhaps unknowingly) rely on an arbitrary generic value.

### Offering multiple IRC servers at alternate ports on the same hostname

An STS policy is imposed for all connections to a hostname. This can create issues when attempting to run multiple IRC servers that share a hostname but use different ports.

For example, a single host might run a production IRC server which advertises an STS policy and another, insecure IRC server on a different port for testing purposes. STS capable clients will lose the ability to connect to the testing server once an STS policy is established for the production server.

To avoid these issues, servers can be offered on separate hostnames, perhaps using subdomains.

## Relationship with other specifications

This section is non-normative.

### Relationship with STARTTLS

STARTTLS is a mechanism for upgrading a connection which has started out as an insecure connection to a secure connection without reconnecting to a different port. This means a server can offer both insecure and secure connections on the same port for compatible clients at the cost of more complex implementations in both clients and servers.

In practice, switching protocols in the middle of the stream has proven to be complicated enough that only a small number of clients bothered implementing STARTTLS.

STS expects that servers instead offer a port that directly services secure connections and it is incompatible with servers that offer secure connections only via STARTTLS on an insecure port.

### Relationship with the `tls` capability

The `tls` capability is a hint for clients that secure connection support is available via STARTTLS.

STS is an improved solution and should be considered a replacement for multiple reasons.

* STS is not merely a hint but requires conforming clients to switch to a secure connection. This means that bookmarked servers are upgraded to use secure connections, even if the bookmark was originally set up to use insecure connections.

* STS mandates that clients only attempt secure connections once a policy has been established, reducing the possibility of users being fooled into allowing a man-in-the-middle attack by downgrading their secure connections to insecure ones. The `tls` capability has no such provision.

* STS is more flexible, as server administrators can configure the lifetime of a policy. 

## Examples

### Redirecting to a secure port with an upgrade policy

A server tells a client connecting insecurely to connect securely on port 6697.

      Insecure Client: CAP LS 302
               Server: CAP * LS :sts=port=6697

After the exchange, the client disconnects and reconnects securely to the same hostname on port 6697.

### Setting an STS persistence policy with a duration

A server tells a secure client to only use secure connections for roughly 6 months.

    Secure Client: CAP LS 302
           Server: CAP * LS :sts=duration=15552000

Until the policy expires:

* The client will continue use the port it is currently connected on connect securely in future.
* It will not make any insecure connection attempts to the same hostname.

### Ignoring an invalid request

A server tells an insecure client to use secure connections for roughly 6 months. There is no port advertised.

      Insecure Client: CAP LS 302
               Server: CAP * LS :sts=duration=15552000

The client ignores this because it has received an STS persistence policy over an insecure connection and the STS cap doesn't contain an upgrade policy.

### Handling tokens with unknown keys

A server tells a secure client to use secure connections for roughly a year, but the value of the STS capability also contains tokens the client doesn't recognise.

    Secure Client: CAP LS 302
           Server: CAP * LS :sts=unknown,duration=31536000,foo=bar

The client ignores the keys it does not understand and until the policy expires:

* The client will use the port it is currently connected to in the future to connect securely.
* It will not make any insecure connection attempts to the same host.

### Handling 0 `duration`

A server tells a client that is already connected securely to remove the STS policy immediately.

    Secure Client: CAP LS 302
           Server: CAP * LS :sts=duration=0

If the client has an STS policy stored for the host it clears the policy. Future attempts to connect insecurely will be allowed.

### Rescheduling expiry on disconnect

A client securely connects to a server, which advertises an STS policy.

    Secure Client: CAP LS 302
           Server: CAP * LS :multi-prefix sts=duration=2592000

The client saves the policy and notes that it will expire in 2592000 seconds (roughly one month). It completes registration, then proceeds as usual.

After 48 hours, the client disconnects.

    Secure Client: QUIT :Bye

The policy is still valid, so the client reschedules expiry for 2592000 seconds from the time of disconnection.

### Updating an STS policy with CAP NEW

A server updates an sts policy by sending a CAP NEW notification.

    Server: CAP * NEW :sts=duration=31536000

If the client has an STS policy stored for the host it updates the policy to expire after 31536000 seconds. Otherwise a new policy is saved for the server.

### Removing an STS policy with CAP NEW

A server removes an sts policy by sending a CAP NEW notification.

    Server: CAP * NEW :sts=duration=0

If the client has an STS policy stored for the host it clears the policy. Future attempts to connect insecurely will be allowed.

### Receiving CAP DEL

A server makes an invalid attempt to remove an sts policy by disabling the CAP.

    Server: CAP * DEL :sts

The client will ignore this attempt and only rely on receiving a duration of 0 to disable STS policies.


### Enabling an STS preload policy

A client securely connects to a server, which advertises an STS policy and opts in to preload lists.

    Secure Client: CAP LS 302
           Server: CAP * LS :sts=duration=2592000,preload


## Errata

New versions of this specification add the `if-host-match` and `port-if-match` keys, which allows some installations to not advertise the `port` key.

Previous versions of this specification only implied that clients ignore poicies that do not include required keys. The specification now explicitly states that clients ignore these invalid policies.
