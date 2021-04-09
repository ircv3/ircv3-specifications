---
title: Forward Mode
layout: spec
meta-description: A channel mode that forwards unwanted users to a different channel
copyrights:
  -
    name: "Val Lorentz"
    period: "2021"
    email: "progval+ircv3@progval.net"
---

## Introduction
This is a standardised syntax to let channel operators redirect
unwanted users to a different channel

## The `FORWARD` ISUPPORT token

Servers publishing the `FORWARD`
[ISUPPORT](https://modern.ircdocs.horse/#feature-advertisement)
token let clients set a channel mode to redirect users to a different
channel when they try to join, by setting a channel mode.
The value of the `FORWARD` token is the mode character which is used to enable
this (e.g. `FORWARD=f` or `FORWARD=L`).

If the server does not support a forward mode, it MAY include the
token with no value (`FORWARD=`).

## The FORWARD mode

Servers MAY allow some users to set the FORWARD mode on a channel.
This mode takes an argument, that is a target channel to forward
users to.

When a client cannot join a channel for whatever reason (user limit,
invite-only channel, banned, ...),
the server MAY send `ERR_LINKCHANNEL` instead of whatever other error numeric
they would have get if the FORWARD mode was not set (`ERR_CHANNELISFULL`,
`ERR_INVITEONLYCHAN`, `ERR_BANNEDFROMCHAN`, ...).

After sending `ERR_LINKCHANNEL` to a user, a server SHOULD proceed as if
the user sent a `JOIN` to the new channel.


## Numerics used by this extension

`470` aka `ERR_LINKCHANNEL` is sent when a user is forwarded to a different
channel than the one they joined.
Syntax:

```
:server 470 <nick> <channel_forwarded_from> <channel_forwarded_to> :<message>
```


## Recursion and loops

When users are redirected to a new channel, servers should check the user
has the right to join it, as if they joined it themselves.
This means they may be redirected to yet an other channel, receive an other
`ERR_LINKCHANNEL`, etc.

For this reason, servers may choose to stop the forwarding any time they wish.

They should send a human-readable `NOTICE` to the user telling them about it.


## Extensions

This section is not normative.

Servers may choose to ignore forwardings for any reason of their choosing.
These reasons include: limiting resource usage, restrictions on when a channel
can set a FORWARD to an other channel, etc.

Servers may also check the target channel exists when a user sets the
FORWARD mode and reply with an appropriate numeric, such as `ERR_INVALIDMODEPARAM`.


## Example

This section is not normative.

With `FORWARD=L`.

User C1 sets the forward mode on two channels, in a "chain":

```
C1: MODE #channel1 :+iL #channel2
 S: :nick1!user!host MODE #channel1 :+iL #channel2
C1: MODE #channel2 :+iL #channel3
 S: :nick1!user!host MODE #channel2 :+lL 1 #channel3
```

User C2 tries to join th first channel of the chain:

```
C2: JOIN #channel1
 S: :server 470 nick2 #channel1 #channel2 :[Link] Cannot join channel #nick1 (channel is invite only) -- transferring you to #channel2
 S: :server 470 nick2 #channel2 #channel3 :[Link] Cannot join channel #nick2 (channel has become full) -- transferring you to #channel3
 S: :nick2!user@host JOIN :#channel3
 S: :server 353 nick2 = #channel3 :nick2 @nick1
 S: :server 366 nick2 #channel3 :End of /NAMES list.
```

