---
title: IRCv3.1 `extended-join` Extension
layout: spec
copyrights:
  -
    name: "Kiyoshi Aman"
    period: "2011"
    email: "kiyoshi.aman@gmail.com"
---
# IRCv3.1 `extended-join` Extension

The extended-join capability extends the JOIN message to include the
account name, or a placeholder if the user hasn't identified with
services. This capability MUST be referred to as 'extended-join' at
capability negotiation time.

When enabled, the JOIN message will designate the account name of the
user when they join a channel.

The JOIN message is one of the following:

    :nick!user@host JOIN #channelname accountname :Real Name

This message represents that the user identified by nick!user@host has
logged in to an acount prior to channel ingress. The penultimate
parameter is the display name of that account. The last parameter is
the user's GECOS.

    :nick!user@host JOIN #channelname * :Real Name

This message represents that the user has not logged in to an account
prior to channel ingress. As the penultimate parameter is an asterisk,
this means that an asterisk is not a valid account name (which it is
not in P10 or TS6 or ESVID).

Please see [`account-notify`](account-notify-3.1.html) for information
on how to take advantage of this capability.
