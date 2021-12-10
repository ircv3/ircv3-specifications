---
title: DNS SRV records
layout: spec
copyrights:
  -
    name: "Simon Ser"
    period: "2021"
    email: "contact@emersion.fr"
---

## Introduction

IRC servers typically use a sub-domain so that IRC connections can be routed
separately from the rest of the services. For instance, an IRC network using
the domain name `example.org` may ask its users to connect to `irc.example.org`
and use DNS load balancing to spread incoming connections to multiple servers.

Unfortunately, this means that users need to make sure to connect to
`irc.example.org` instead of `example.org` as the latter won't accept IRC
connections.

Additionally, other connection details such as the port and TLS usage might
differ from one network to another.

Other services (e.g. IMAP, SMTP) have standardized DNS SRV records which can be
used to discover connection details. This specification defines DNS SRV records
for IRC.

SRV records are defined in [RFC 2782].

## Description

IRC networks conforming to this specification MUST publish an SRV record with
the "ircs" service label. The record identifies an IRC server where TLS is
initiated directly upon connection to the server.

Example:

    _ircs._tcp SRV 0 1 6697 irc.example.org.

There is no defined service label for unencrypted connections.

[RFC 2782]: https://datatracker.ietf.org/doc/html/rfc2782
