# SASL authentication in IRC v3.2

Copyright (c) 2014 Attila Molnar <attilamolnar@hush.com>

Unlimited redistribution and modification of this document is allowed
provided that the above copyright notice and this permission notice
remains in tact.

## Description

This document describes how [SASL authentication](extensions/sasl-3.1)
makes use of the
[capability negotiation protocol updates introduced in 3.2](specification/capability-negotiation-3.2.md).

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
