---
title: IRCv3 `account-notify` Extension
layout: spec
redirect_from:
  - /specs/extensions/account-notify-3.1.html
copyrights:
  -
    name: "William Pitcock"
    period: "2010"
    email: "nenolod@dereferenced.org"
---
The account-notify client capability allows a client to be notified
when a client's account name changes.  This capability MUST be
referred to as 'account-notify' at capability negotiation time.

When enabled, clients will get the ACCOUNT message to designate the
account name changes for clients on common channels with them, including
themselves.

The ACCOUNT message is one of the following:

    :nick!user@host ACCOUNT accountname

This message represents that the user identified by nick!user@host has
logged into a new account.  The last parameter is the display name of
that account.

    :nick!user@host ACCOUNT *

This message represents that the user identified by nick!user@host has
logged out of their account.  As the last parameter is an asterisk, this
means that an asterisk is not a valid account name (which it is not in P10
or TS6 or ESVID).

In order to take full advantage of the ACCOUNT message, WHOX must be
supported by the ircd.  In this case, the appropriate strategy to ensuring
you always have the account name to display is to do the following:

 1. Enable the "account-notify" capability at capability negotiation time
    during the login handshake.

 2. When joining a channel, query the channel using WHO and ensure that you
    include the `a` format token in your WHOX token request.  When you get a
    reply, do appropriate caching.

 3. If the extended-join capability is available, enable it at client
    capability negotiation time during the login handshake, and then set the
    account name based on what is sent in the extended JOIN command.

    Otherwise, if extended-join is unavailable: When new users join a channel
    that your client does not know the account name for yet, do a WHO query
    against that client, again including the `a` format token in your WHOX
    token request field.  When you get a reply, do appropriate caching.

 4. When the client receives an ACCOUNT message, update the account name for the
    client in question in your account name cache.

## Errata

A previous version of this specification advised making a WHO request following a netsplit.
This advice was deemed unnecessary and was removed. See
[issue #146](https://github.com/ircv3/ircv3-specifications/issues/146) for more information, including
rationale for this clarification.

A previous version of this specification didn't make it clear that account-notify messages are sent for
a client's own account name changes.