invite-notify client capability specification
---------------------------------------------

Copyright (c) 2013 Adam <Adam@anope.org>

Copyright (c) 2014 Attila Molnar <attilamolnar@hush.com>

Unlimited redistribution and modification of this document is allowed
provided that the above copyright notice and this permission notice
remains in tact.

## Description

The `invite-notify` client capability allows a client to specify that it
would like to be notified when users are invited to channels.

The name of this client capability MUST be named `invite-notify`.

This capability is designed to replace the traditional "X has invited
Y to #chan" server notices, and to provide a standard way that allows
clients to learn when another client does an /INVITE.

## Format

The format of the INVITE message is as follows:

    :<inviter> INVITE <target> <channel>

The message source, `inviter` is the user doing the INVITE, target
is the user being invited, `channel` is the channel where `target` is
being invited.

## Example

    :ChanServ!ChanServ@example.com INVITE Attila #channel

This message translates to ChanServ has invited Attila to #channel.

## Notes

The server is not required to send the INVITE message described in
this document to all clients supporting this capability on a channel.
For example, a server may choose to only send the message to clients
having channel op, or only to clients that have the privilege to do
an /INVITE on the channel themselves.
