---
title: IRCv3 TLS Best Practices
layout: spec
copyrights:
  -
    name: "William Pitcock"
    period: "2016"
    email: "nenolod@dereferenced.org"
---

This document specifies a series of baseline security practices for managing TLS on IRC networks.
It should be considered non-normative.


# Clients

* Clients SHOULD upgrade the default port to 6697 and enable TLS per IETF [RFC 7194][rfc7194].

  [rfc7194]: https://tools.ietf.org/html/rfc7194

* Clients SHOULD allow a user to downgrade to plaintext if they explicitly configure the client
  to do so.  A suggested `/server` command may be something like:

    /server -insecure irc.example.com 6667

* Clients SHOULD implement [STS][sts-3.3], with the following suggestions:

  * Clients SHOULD implement the ability to explicitly disable STS for a connection if the user
    explicitly specifies that they wish to use plaintext.

  * Clients SHOULD implement the ability to flush the STS cache to allow correction of any
    incorrect STS policy.

  [sts-3.3]: http://ircv3.net/specs/core/sts-3.3.html

* Clients SHOULD allow the user to generate a client certificate and use that as an authentication
  token via the SASL EXTERNAL mechanism.

* Clients SHOULD NOT provide any feedback that a TLS connection is secured, unless the user has
  explicitly pinned the server certificate to their trust store.

* Clients SHOULD provide access to the underlying TLS implementation's trust store, to allow for
  registration and access of client certificate data stored on smartcards, pinning certificates,
  etc.

* Clients MAY use a conservative CA certificate bundle.  System certificate bundles tend to
  include certificate authorities which are completely inappropriate for signing IRC-related
  certificates.


# Servers

* Server certificates SHOULD NOT be self-signed, and instead SHOULD be anchored to a reasonable
  trust root.

* Servers SHOULD publish an STS policy, when there is no explicit configuration either way, if
  the server software can verify that the certificate is attached to a trusted CA root in the
  server's CA trust root.

* Server admins for large networks SHOULD chain all servers to a common CA root, which can be
  anchored by the user as a trust root.  This common CA root may be cross-signed by another CA
  for compatibility with system trust stores.

* As networks migrate to TLS being the primary option, server software SHOULD instead highlight
  insecure users instead of secure ones.  Examples include:

  - adding a channel mode that allows insecure users to join (reverse of SSL-only channel mode)
  - inclusion of /WHOIS output indicating the user is not using security

* If a network is moving to TLS-only, the network MAY provide a plaintext listener that provides
  migration instructions, as well as an STS policy for interested clients to follow.
