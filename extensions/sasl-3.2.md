---
title: IRCv3.2 SASL Authentication
layout: spec
copyrights:
  -
    name: "Attila Molnar"
    period: "2014"
    email: "attilamolnar@hush.com"
  -
    name: "William Pitcock"
    period: "2015"
    email: "nenolod@dereferenced.org"
---
## Description

This document describes how [SASL authentication](/specs/extensions/sasl-3.1.html)
makes use of the
[capability negotiation protocol updates introduced in 3.2](/specs/core/capability-negotiation-3.2.html),
and adds support for reauthentication if authentication is lost via the [cap-notify](/specs/extensions/cap-notify-3.2.html)
capability.

## Advertisement of `sasl` capability when authentication is unavailable

Servers MUST NOT advertise the `sasl` capability if the authentication layer is
unavailable.

Servers MUST `NAK` any `sasl` capability request if the authentication layer is
unavailable.

If a client requires pre-authentication and is unable to obtain the `sasl` capability,
then the client MUST disconnect and MAY retry the connection.

## Mechanism list in CAP LS

Servers SHOULD advertise the available SASL authentication mechanisms in the
value of the `sasl` client capability in `CAP LS`.

The value, if present, MUST be a comma (`,`) separated list of SASL
authentication mechanisms.

Clients MUST handle the case when there is no value for the `sasl` capability
in `CAP LS` even though they are attempting to use a version of the capability
negotiation protocol that allows values (i.e. 3.2).

Clients MUST handle the case when the value contains one or more mechanisms
they do not understand.

## Integration with `cap-notify`

The `sasl` capability is integrated with the new `cap-notify` framework, such that
if `cap-notify` is an active capability, the client will be notified about status
changes concerning the availability of `sasl` authentication.

Servers MUST advertise availability of the `sasl` capability to any clients which have
requested the `cap-notify` notification.

## SASL Reauthentication

Servers MUST allow a client, authenticated or otherwise, to reauthenticate by
sending a new `AUTHENTICATE` message at any time.

Servers MAY disconnect ANY client at any time as a result of failed authentication,
including both unregistered and registered clients, but MUST provide the reason
for the authentication failure prior to disconnection.

## Usage

Clients SHOULD pick a mechanism present in the `CAP LS` reply they get from
the server and attempt to use that mechanism for authentication after they
request the `sasl` capability.

Clients MUST handle the case when a mechanism they picked from the list turns
out to be not available.

## Examples

Example where the server only supports the EXTERNAL mechanism:

    Client: CAP LS 302
    Server: CAP * LS :sasl=EXTERNAL

Example where the server supports multiple authentication mechanisms and the client
picks its favorite and attempts to use it:

    Client: CAP LS 302
    Client: NICK modernclient
    Client: USER modernclient 0 0 :Modern Client
    Server: CAP * LS :sasl=EXTERNAL,FOO,DH-AES,BAR,DH-BLOWFISH,FOOBAR,PLAIN batch cap-notify
    Client: CAP REQ :sasl
    Server: CAP modernclient ACK :sasl
    Client: AUTHENTICATE PLAIN
    Server: AUTHENTICATE +
    Client: AUTHENTICATE ...

Example where the server later adds support for authentication, such as regaining
access to an authentication server after a netsplit:

    Server: CAP modernclient NEW :sasl=PLAIN
    Client: CAP REQ :sasl
    Server: CAP modernclient ACK :sasl
    Client: AUTHENTICATE PLAIN
    Server: AUTHENTICATE +
    Client: AUTHENTICATE ...

Example where the server disconnects a client for excessive reauthentications:

    Client: CAP REQ :sasl
    Server: CAP modernclient ACK :sasl
    Client: AUTHENTICATE PLAIN
    Server: AUTHENTICATE +
    Client: AUTHENTICATE ...
    Server: :irc.server.tld 904 modernclient :SASL authentication failed
    Client: AUTHENTICATE PLAIN
    Server: AUTHENTICATE +
    Client: AUTHENTICATE ...
    Server: :irc.server.tld 904 modernclient :SASL authentication failed
    Client: AUTHENTICATE PLAIN
    Server: AUTHENTICATE +
    Client: AUTHENTICATE ...
    Server: :irc.server.tld 904 modernclient :SASL authentication failed
    Server: ERROR :Too many SASL authentication failures
