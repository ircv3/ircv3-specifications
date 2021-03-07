---
title: IRCv3 bouncer extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Darren Whitlen"
    period: "2018-2021"
    email: "darren@kiwiirc.com"
---

## BOUNCER

This feature is intended to provide an API for clients to build a UI around bouncer IRC servers, to manage the users IRC networks and buffers in a user friendly way and to keep multiple clients and devices synced together.

It is based around network and buffer entities which both have key/value tags, such as `joined=true` for a buffer, with commands to manage these. A list of common tags can be found under the tags section of this document.

From a single bouncer connection, a complete implementation will allow users to:
* Add, remove, modify and search networks under their bouncer account.
* List and search channels and queries.


## Notes
References in this document:
- A `network` is an IRC network that a bouncer connects to
- A `buffer` is either a channel or query
- A client is a users client or device that connects to the bouncer

A network is identified by a per-user unique ID that MUST not change. A human readable label MUST also be associated with the network that the user can identify it with in a client UI - this is the `network` tag in most commands. The ID and label may be the same as long as the ID does not change. Neither the ID or label may contain a space or `:` characters.

All commands and data for this bouncer feature MUST be associated to a logged-in and authorised user. It is the implementors choice on how the user authentication is handled and is outside the scope of this document.

## Capabilities

This adds a capability called `BOUNCER`. Once negotiated the server may change it's behaviour to better suit bouncer clients, such as not automatically sending `JOIN` commands for already joined channels.

## RPL_ISUPPORT
This adds an isupport token called `BOUNCER`. Its optional value is message-tag encoded data including:
* `network=` The name or label of the network being connected to.
* `netid=` The ID of the network being connected to.

Eg: `BOUNCER=network=freenode;netid=1234`

## `BOUNCER` command
This introduces the `BOUNCER` command, followed by a case insensitive subcommand.

All tags sent and received by this command are message-tags encoded.

***Note***: Where a network ID is used to specify a network in a command, a `*` character may be used instead to signify the active network the client is connected to.


### Connecting a network
Syntax: `[c] BOUNCER connect <netid>`

Replies:
* Various errors:
  - `[s] BOUNCER connect * :Invalid arguments given`
  - `[s] BOUNCER connect netid ERR_NETNOTFOUND`

The connection state of the network will be received as mentioned in the network state section.

### Disconnecting a network
Syntax: `[c] BOUNCER disconnect <netid> [:quit message]`

Replies:
* Various errors:
  - `[s] BOUNCER disconnect * ERR_INVALIDARGS`
  - `[s] BOUNCER disconnect netid ERR_NETNOTFOUND`

The connection state of the network will be received as mentioned in the network state section.


### Listing networks
Syntax: `[c] BOUNCER listnetworks [*]`

This command MUST return all networks under the users bouncer account. The optional second parameter of `*` is a wildcard search string that filters the returned networks. It is matched against the `network` tag of each network.

This may be used for rudimentry network organisation, Eg. `work/*` or `social/*` where networks are labelled with `work/somenetwork` and `social/rizon` structures.

The end of the network list MUST be indicated by an `RPL_OK` message, even if there are no networks to list. Eg:
~~~
[s] BOUNCER listnetworks netid network=freenode;host=irc.freenode.net;port=6667;state=disconnected;nick=bob;
[s] BOUNCER listnetworks netid network=IRC.com;host=irc.irc.com;port=6697;state=connected;tls=1;nick=bob;
[s] BOUNCER listnetworks RPL_OK
~~~

A complete list of tags for a network can be found in the tags section of this document.


### Listing buffers
Syntax: `[c] BOUNCER listbuffers <netid | *>`

This command MUST return all active buffers. An active buffer will be defined by the implementation, for example, non-archived buffers, joined channels, active queries, or any mix.

Replies:
* Various errors:
  - `[s] BOUNCER listbuffers * ERR_INVALIDARGS`
  - `[s] BOUNCER listbuffers * ERR_NETNOTFOUND`

The end of the buffer list MUST be indicated by an `RPL_OK` message, even if there are no buffers to list. Eg:
  ~~~
  [s] BOUNCER listbuffers netid network=freenode;buffer=#chan;joined=1;topic=some\stopic
  [s] BOUNCER listbuffers netid network=freenode;buffer=somenick;
  [s] BOUNCER listbuffers netid RPL_OK
  ~~~

A complete list of tags for a buffer can be found in the tags section of this document.

### Removing a buffer
Syntax: `[c] BOUNCER delbuffer <netid | *> <buffername>`

This command will delete a buffer from the bouncer. The server MAY send a `LISTBUFFERS` listing to all the users connected clients after it has been deleted to keep all clients in sync.

Replies:
* Buffer deleted:
  - `[s] BOUNCER delbuffer netid buffername RPL_OK`
* Various errors:
  - `[s] BOUNCER delbuffer * * ERR_INVALIDARGS`
  - `[s] BOUNCER delbuffer * * ERR_NETNOTFOUND`


### Changing a buffer
Syntax: `[c] BOUNCER changebuffer <netid | *> <buffername> seen=2018-01-25T17:00:00Z`

This command is used to change a tag value for a buffer. The tags are message-tags encoded. A complete list of tags for a buffer can be found in the tags section of this document.

The `seen` tag may be `1` to mark the buffer as seen at the current time.

The server MAY send a `LISTBUFFERS` listing to all the users connected clients after it has been deleted to keep all clients in sync.

Replies:
* Buffer changed:
  - `[s] BOUNCER changebuffer netid buffername RPL_OK`
* Various errors:
  - `[s] BOUNCER changebuffer * * ERR_INVALIDARGS`
  - `[s] BOUNCER changebuffer * * ERR_NETNOTFOUND`
  - `[s] BOUNCER changebuffer netid buffername ERR_BUFFERNOTFOUND`
  - `[s] BOUNCER changebuffer netid buffername ERR_UNKNOWN :Optional extra information`


### Adding a network
Syntax: `[c] BOUNCER addnetwork network=freenode;host=irc.freenode.net;port=6667;nick=prawnsalad;user=prawn`

This command adds a new network to the bouncer. A complete list of tags for a network can be found in the tags section of this document.

The bouncer MAY reject this new network for any reason and MUST then reply with an error message as shown below. If accepted, the bouncer MUST generate a new unique ID for this new network under the users account. The bouncer MAY also include default values for any tags if any are not provided.

The bouncer MAY send the `LISTNETWORKS` reply to all the users connected clients to keep them in sync.

Replies:
* Network addeed:
  - `[s] BOUNCER addnetwork netid freenode RPL_OK`
* Various errors:
  - `[s] BOUNCER addnetwork * * ERR_NEEDSNAME`
  - `[s] BOUNCER addnetwork * freenode ERR_NAMEINUSE`
  - `[s] BOUNCER addnetwork * freenode ERR_INVALIDPORT`
  - `[s] BOUNCER addnetwork * freenode ERR_MAXNETWORKS`
  - `[s] BOUNCER addnetwork * freenode ERR_UNKNOWN :Optional extra information`


### Changing a network
Syntax: `[c] BOUNCER changenetwork <netid | *> host=irc.freenode.net;port=6667;nick=prawnsalad;user=prawn`

Change an existing network on the bouncer. The tags are the same as mentioned in the `addnetwork` command. At least one tag MUST be given.

The bouncer MAY send the `LISTNETWORKS` reply to all the users connected clients to keep them in sync.

Replies:
* Network changed:
  - `[s] BOUNCER changenetwork netid RPL_OK`
* Various errors:
  - `[s] BOUNCER changenetwork * ERR_INVALIDARGS`
  - `[s] BOUNCER changenetwork netid ERR_NETNOTFOUND`
  - `[s] BOUNCER changenetwork netid ERR_INVALIDPORT`
  - `[s] BOUNCER changenetwork netid ERR_UNKNOWN :Optional extra information`
    


### Removing a network
Syntax: `[c] BOUNCER delnetwork <netid | *>`

Delete a network from the bouncer. If deleting the active network the implimentation may choose how to handle the client connection. Examples include closing it or parting all channels and resorting it to a temporary connection.

The bouncer MAY send the `LISTNETWORKS` reply to all the users connected clients to keep them in sync.

Replies:
* Network removed:
  - `[s] BOUNCER delnetwork netid RPL_OK`
* Various errors:
  - `[s] BOUNCER delnetwork * ERR_INVALIDARGS`
  - `[s] BOUNCER delnetwork netid ERR_NETNOTFOUND`
  - `[s] BOUNCER delnetwork netid ERR_UNKNOWN :Optional extra information`


## Network state notifications
As a network state changes on the server such as connecting or disconnection, it MUST send the changes as they happen to any users connected clients.

* Connection state changes:
  - `[s] BOUNCER state netid netname connecting`
  - `[s] BOUNCER state netid netname connected`
  - `[s] BOUNCER state netid netname disconnected`

## Network and buffer tags
This document has described several tags that SHOULD be supported. Implementations MAY introduce other tags but MUST contain a vendor prefix, eg. `kiwibnc/mytag=value`.

#### Network tags
Implementations MUST recognise the following:
- `network` - The human readable name for the network. It may be changed.
- `state` - One of `connected`, `connecting`, or `disconnected` that represents the current connection to the IRC network.
- `host` - The hostname to connect to the IRC network.
- `port` - The port to connect to the IRC network.
- `tls` - `1` if this IRC network requires TLS, `0` if not.
- `tlsverify` - `1` if the bouncer should verify the TLS connection. `0` if not.
- `password` - The server password.
- `sasl_account` - The SASL account name to log into the IRC network with.
- `sasl_pass` - The SASL password to log into the IRC network with.
- `nick` - The nick for the active connection to the IRC network or the nick to use when connecting.
- `username` - The username for the active connection to the IRC network or the username to use when connecting.
- `realname` - The realname for the active connection to the IRC network or the realname to use when connecting.


#### Buffer tags

Implementations MUST recognise the following:
- `network` - The name of the network this buffer belongs to.
- `buffer` - The name of this buffer.
- `seen` - The ISO 8601 timestamp of when the user last saw this buffer. Used to determine the point at which the user as read up to.
- `joined` - Channels only. `1` if joined to the channel, `0` if not joined.
- `topic` - The topic for this buffer. Usually channels only.

Optional tags that MAY be recognised:
- `notify` - One of `message`, `highlight`, or `never` that represents the notification level for this buffer.

### Multi-client considerations

If a server has multiple clients connected for a single user such as a desktop and mobile client, an implementation may want to keep each connected client in sync to have their interfaces in the same state. For this, servers MAY send a `LISTNETWORKS` or `LISTBUFFERS` listing to each connected client as they change.
