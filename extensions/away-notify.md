---
title: "`away-notify` Extension"
layout: spec
redirect_from:
  - /specs/extensions/away-notify-3.1.html
copyrights:
  -
    name: "Keith Buck"
    period: "2012"
    email: "mr_flea@esper.net"
---

## Description

The away-notify client capability allows a client to specify that it
would like to be notified when users are marked/unmarked as away. This
capability is referred to as 'away-notify' at capability negotiation
time.

This capability is designed to replace polling of WHO as a more
efficient method of tracking the away state of users in a channel. The
away-notify capability both conserves bandwidth as WHO requests are
not continually sent and allows the client to be notified immediately
upon a user setting, changing or removing their away state (as opposed to when
WHO is next polled).

When this capability is enabled, clients will be sent an AWAY message
when a user sharing a channel with them sets, changes or removes their away
state, as well as when a user joins and has an away message set.
(Note that AWAY will not be sent for joining users with no away
message set.)

Clients SHOULD NOT be sent AWAY messages to notify them of their own away
status (as they can rely on `RPL_NOWAWAY` and `RPL_UNAWAY`).

The format of the AWAY message is as follows:

    :nick!user@host AWAY [:message]

If the message is present, the user (specified by the nick!user@host
mask) is going away.  If the message is not present, the user is
removing their away message/state.

To fully track the away state of users, clients should:

1) Enable the away-notify capability at negotiation time.

2) Execute WHO when joining a channel to capture the current away
   state of all users in that channel.

3) Update state appropriately upon receiving an AWAY message.

## Erratum

The previous version of this specification did not explicitly say
users should not get notifications of their own away status with AWAY.
This was clarified in 2023
