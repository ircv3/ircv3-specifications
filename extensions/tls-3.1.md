---
title: IRCv3.1 `tls` Extension
layout: spec
copyrights:
  -
    name: "William Pitcock"
    period: "2012"
    email: "nenolod@dereferenced.org"
  -
    name: "Attila Molnar"
    period: "2014"
    email: "attilamolnar@hush.com"
---
## The tls Client Capability

The tls client capability indicates that the server supports the STARTTLS command.

It is worth noting here that a client may request the tls capability with CAP REQ,
but this won't affect that client's ability to later send the STARTTLS command.
The STARTTLS command will always operate on servers which advertise the capability
regardless of if the client requests the capability or not. 

To use STARTTLS a client simply sends the STARTTLS command to the server before the
client registers with the server using the NICK and USER commands, the server then
sends numeric 670 (RPL_STARTTLS), and the client initiates a SSL/TLS handshake.

If there is an error with setting up TLS on the server side, the server must send
numeric 691 (ERR_STARTTLS) containing a human-readable description of the error as
its parameter.

## Recommendations

If the user has not configured the connection to be explicitly plaintext
and the client sees the server advertising the `tls` capability, the
client should ask the user whether they want to use TLS for all future
connections (or until disabled manually). If the user agrees, all future
connections to the server must use TLS via STARTTLS. If TLS cannot be
negotiated, it is a hard error and the client must abort the connection.
This is to make sure that the connection cannot be downgraded to plaintext
by a man-in-the-middle once it has been established that TLS support is
available.

It should be assumed that once a server has advertised TLS support it is not
going to stop supporting it. This is similar to how SSL-only ports are not
expected to go away.

## Numerics used by this extension

`670` aka `RPL_STARTTLS` to indicate that the client may begin the TLS handshake.

    :server 670 <nick> :STARTTLS successful, proceed with TLS handshake

`691` aka `ERR_STARTTLS` to indicate that a server-side error has occured.

    :server 691 <nick> :STARTTLS failed (Wrong moon phase)

_(The final "text" parameter is not to be machine-parsed, as it tends to vary
between implementations and translations.)_
