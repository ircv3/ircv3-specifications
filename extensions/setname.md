---
title: IRCv3 `setname` Extension
layout: spec
copyrights:
  -
    name: "Janne Mareike Koschinski"
    period: "2019"
    email: "janne@kuschku.de"
---

## Motivation

Historically, a user's realname could only be set on the initial connection
handshake. However, multiple IRC servers have provided independent non-standard
commands to update the realname without reconnecting. This specification
describes a standardised behaviour based on these existing implementations.

## Description

The `setname` client capability allows clients to change their realname
(GECOS) on an active connection. It also allows servers to directly inform
clients about such a change.
This avoids a client reconnect when updating this value.

This capability MUST be referred to as `setname` at capability
negotiation time.

Servers advertising the `setname` capability MUST support usage of the
command even while the capability is not negotiated. Clients MUST NOT prevent
users from manually using the command while the capability is not negotiated.

If a client sends a `SETNAME` command without having negotiated the capability,
the server SHOULD handle it silently (with no response), as historic
implementations did.

When enabled, servers MUST provide a `NAMELEN` token in `RPL_ISUPPORT` with the
maximum allowed length for realnames.

The client-to-server `SETNAME` command looks as follows:

    SETNAME :realname goes here

This command represents the intent to change the realname. The only, trailing
parameter is the new realname. If the capability is negotiated, the client
MUST assume that no change has happened until the server confirms this
change.

Servers MUST check the realname for validity (taking length limits into
consideration). If they accept the realname change, they MUST send the
server-to-client version of the `SETNAME` message to all clients in common
channels, as well as to the client from which it originated, to confirm the
change has occurred.

The `SETNAME` message MUST NOT be sent to clients which do not have the
`setname` capability negotiated.

The server-to-client `SETNAME` message looks as follows:

    :nick!user@host SETNAME :realname goes here

This message represents that the user identified by nick!user@host has changed
their realname to another value. The only, trailing parameter is the new
realname.

In order to take full advantage of the `SETNAME` message, clients must be
modified to support it. The proper way to do so is this:

1) Enable the `setname` capability at capability negotiation time during
   the login handshake.

2) On receipt of a server-to-client `SETNAME` message, update the realname
   portion of data structures and process channel users as appropriate.

## Errors

The server MUST use the [standard replies][] extension to notify the client of
failed `SETNAME` commands.

If the server rejects the realname as a result of a validation failure, it MUST
send a `FAIL` message with the `INVALID_REALNAME` code.

    FAIL SETNAME INVALID_REALNAME :Realname is not valid

If the server rejects the change for any other reason, it MUST send a `FAIL`
message with the `CANNOT_CHANGE_REALNAME` code and an appropriate description of
the reason:

    FAIL SETNAME CANNOT_CHANGE_REALNAME :Slow down your realname changes
    FAIL SETNAME CANNOT_CHANGE_REALNAME :Cannot change realname while banned from a channel

## Examples

Complex realname with spaces

    C: SETNAME :Bruce Wayne <bruce@wayne.enterprises>
    S: :batman@~batman!bat.cave SETNAME :Bruce Wayne <bruce@wayne.enterprises>

Simple realname

    C: SETNAME Batman
    S: :batman@~batman!bat.cave SETNAME Batman

Example with realname rejected by the server

    C: SETNAME :Heute back ich, morgen brau ich, übermorgen hol ich der Königin ihr Kind; ach, wie gut, dass niemand weiß, dass ich Rumpelstilzchen heiß!
    S: FAIL SETNAME INVALID_REALNAME :Realname is not valid

[standard replies]: ../extensions/standard-replies.html