---
title: IRCv3 `setname` Extension
layout: spec
copyrights:
  -
    name: "Janne Mareike Koschinski"
    period: "2019"
    email: "janne@kuschku.de"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `setname` tag name. Instead, implementations SHOULD use
the `draft/setname` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Motivation

Realnames have historically been immutable for a connection, which has
historically been accepted, as they were rarely used and usually not as present
in client UIs. Still, multiple IRCds have provided independent, non-standard
commands to update realnames.

Nowadays, multiple clients show realnames directly in chat, next to the nick
name, or use information from the realname to provide enhancements, such as
using emails in the realname to provide gravatar based avatars, leading to
higher demand for this functionality than ever.

## Description

The `draft/setname` client capability allows clients to change their realname
(GECOS) as well as allowing servers to directly inform clients about such a
change without having to send a fake quit and join and without clients having
to actually disconnect and reconnect. 

This capability MUST be referred to as `draft/setname` at capability
negotiation time.

When enabled, clients are allowed to send a `SETNAME` command to notify servers
of their intent to update their realname.

This client-to-server `SETNAME` command looks as follows:

    SETNAME :realname goes here

This command represents the intent to change the realname. The only, trailing
parameter is the new realname. Until the server confirms this change, the
client MUST assume that no change has happened yet.

Servers MUST check the realname for validity (taking length limits into
consideration) and, if they apply the realname change, MUST send the
server-to-client version of the `SETNAME` message to all clients in common
channels, as well as to the client from which it originated.

This server-to-client `SETNAME` message looks as follows:

    :nick!user@host SETNAME :realname goes here

This message represents that the user identified by nick!user@host has changed
their realname to another value. The only, trailing parameter is the new
realname.

In order to take full advantage of the `SETNAME` message, clients must be
modified to support it. The proper way to do so is this:

1) Enable the `draft/setname` capability at capability negotiation time during
   the login handshake.

2) Update the realname portion of data structures and process channel users as
   appropriate.

## Examples

**Client-to-Server**

Complex realname with spaces

    SETNAME :Bruce Wayne <bruce@wayne.enterprises>
    
Simple Realname

    SETNAME Batman

**Server-to-Client**

Complex realname with spaces

    :batman@~batman!bat.cave SETNAME :Bruce Wayne <bruce@wayne.enterprises>
    
Simple Realname

    :batman@~batman!bat.cave SETNAME Batman
