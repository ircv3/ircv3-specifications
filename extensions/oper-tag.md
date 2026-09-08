---
title: "`oper-tag` Extension"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "David Schultz"
    period: "2022"
    email: "me@zpld.me"
  -
    name: "Sadie Powell"
    period: "2026"
    email: "sadie@sadiepowell.dev"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `oper` tag name. Instead, implementations SHOULD use the `draft/oper` tag name to be interoperable with other software implementing a compatible work-in-progress version.

## Description

The `draft/oper-tag` capability causes the server to add the `draft/oper` and `draft/oper-role` [message tags][] to messages sent by a user who is currently an IRC operator.

The `draft/oper` tag marks a user as an IRC operator. The value of this tag, if specified, MUST be the operator's name (e.g. from the `<name>` parameter of the `OPER <name> <password>` command).

The `draft/oper-role` tag shows the role that a IRC operator has on a network (e.g. NetAdmin). This role is purely descriptive and has no consistent meaning between networks. This tag MUST only be sent on messages that have the `draft/oper`tag.

Servers supporting this capability MAY be configured to restrict visibility of the tags or their values. For users who can see the tags they MUST be added by the IRC server to all commands sent by a user (e.g. `PRIVMSG`, `MODE`, `NOTICE`, etc) and SHOULD be added to any numeric replies sent on behalf of the user (e.g. `RPL_WHOSPCRPL`).

## Example

Consider that I am identified to the `launchd` operator account while chatting with the `lunchd` nickname. I will send messages to four users with different configured levels of visibility.

    @draft/oper=launchd :lunchd PRIVMSG full_access_friend :it's me!
    @draft/oper :lunchd PRIVMSG partial_access_friend :you can see I'm an oper
    @draft/oper;draft/oper-role=NetAdmin :lunchd PRIVMSG partial_access_friend2 :you can see I'm an oper and my role
    :lunchd PRIVMSG unprivileged_friend :don't mind me

[message tags]: ../extensions/message-tags
