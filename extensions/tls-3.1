# tls Client Capability Extension

Copyright (c) 2012 William Pitcock <nenolod@atheme.org>.

Copyright (c) 2014 Attila Molnar <attilamolnar@hush.com>.

Unlimited redistribution and modification of this document is allowed provided
that the above copyright notice and this permission notice remains in tact.

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
it's parameter.

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
going to stop supporting it. This is similiar to how SSL-only ports are not
expected to go away.
