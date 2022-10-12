---
title: "`account-tag` Extension"
layout: spec
redirect_from:
  - /specs/extensions/account-tag-3.2.html
copyrights:
  -
    name: "Mantas Mikulėnas"
    period: "2013-2015"
    email: "grawity@gmail.com"
---

## Description

The `account-tag` capability causes the server to add a [message tag][] containing
the command sender's services account to commands sent to the client that has
requested this capability. It supersedes the `identify-msg` extension.

The tag MUST be named `account`, and contain the sender's current services
username. (If the user is not identified to any services account, the tag MUST
NOT be sent.)

The tag MUST be added by the ircd to all commands sent by a user (e.g. PRIVMSG,
MODE, NOTICE, and all others). The tag also SHOULD be added by the ircd to all
numerics directly caused by the sender. For example, if the target user has
"caller ID" enabled (e.g. user modes +R or +g in Charybdis), the tag SHOULD be
added to numerics indicating blocked message attempts (numeric 718 aka
`RPL_UMODEGMSG`).

Example (demonstrated using the `account-notify` capability):

    <-- :user PRIVMSG #atheme :Hello everyone.
    <-- :user ACCOUNT hax0r
    <-- @account=hax0r :user PRIVMSG #atheme :Now I'm logged in.
    <-- @account=hax0r :user ACCOUNT bob
    <-- @account=bob :user PRIVMSG #atheme :I switched accounts.

## Relationship to other extensions

Rationale: This extension was proposed because `identify-msg` does not always
provide enough information (e.g. the user may be logged into a different
account); `extended-join` and `account-notify` do not work with private
messages (unless both participants share at least one channel) or with messages
sent from outside users to a `-n` channel; and `metadata` makes the client
actively request someone's account name (i.e. another client may send a private
message and quickly switch accounts before the local client even has a chance
to send a metadata query).

This extension supersedes `identify-msg`. This extension does not deprecate
`extended-join`, as the latter also extends JOIN to send more information than
just the account name. Similarly, this extension does not deprecate
`account-notify`, as the latter provides real-time notifications while this
extension does not.

[message tag]: ../extensions/message-tags.html

## Errata

* Earlier version of this specification said that account tags MUST be added to numerics that were directly caused by a specific user. This was relaxed to SHOULD after it was found to be widely not implemented and a problem for existing implementations to implement.
