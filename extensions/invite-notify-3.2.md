---
title: IRCv3.2 `invite-notify` Extension
layout: spec
copyrights:
  -
    name: "Adam"
    period: "2013"
    email: "adam@anope.org"
  -
    name: "Attila Molnar"
    period: "2014"
    email: "attilamolnar@hush.com"
---
## Description

The `invite-notify` client capability allows a client to specify that it
would like to be notified when users are invited to channels.

This client capability MUST be named `invite-notify`.

This capability is designed to replace the traditional "X has invited
Y to #chan" server notices, and to provide a standard way that allows
clients to learn when another client does an /INVITE.

## Format

The format of the INVITE message is as follows:

    :<inviter> INVITE <target> <channel> [status]

The message source, `inviter` is the user doing the INVITE, target
is the user being invited, `channel` is the channel where `target` is
being invited. `status` is an optional status character, which MUST be
present in the `STATUSMSG` token in the server's `RPL_ISUPPORT` (005)
values if used.

## Example

    :ChanServ!ChanServ@example.com INVITE Attila #channel

This message translates to ChanServ has invited Attila to #channel.

    :ChanServ!ChanServ@example.com INVITE Alissa #channel @

This message translates to ChanServ has invited Alissa to #channel,
but only shows the message to operators with status of @ or higher.

## Notes

The server is not required to send the INVITE message described in
this document to all clients supporting this capability on a channel.
For example, a server may choose to only send the message to clients
having channel op, or only to clients that have the privilege to do
an /INVITE on the channel themselves. In this case, the server MAY 
use an appropriate `STATUSMSG` status on the channel argument.

## Errata

Added a mention of the `STATUSMSG` argument.
