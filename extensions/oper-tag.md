---
title: "`oper-tag` Extension"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "David Schultz"
    period: "2022"
    email: "me@zpld.me"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `oper` tag name. Instead, implementations SHOULD use the
`draft/oper` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

## Description

The `draft/oper` capability causes the server to add a [message tag][] to messages sent by a user who is currently an IRC operator.

The tag MUST be named `draft/oper`. The value of the tag, if specified, MUST identify the operator.

The tag MUST be added by the ircd to all commands sent by a user (e.g. PRIVMSG,
MODE, NOTICE, and all others) and numeric replies sent on behalf of the user.

Servers supporting this capability MAY be configured to restrict visibility of this tag or its value.

## Example

Consider that I am identified to the `launchd` operator account while chatting with the `lunchd` nickname. I will send messages to three users with different configured levels of visibility.

    @draft/oper=launchd :lunchd PRIVMSG full_access_friend :it's me!
    @draft/oper :lunchd PRIVMSG partial_access_friend :you can see I'm an oper
    :lunchd PRIVMSG unprivileged_friend :don't mind me

[message tag]: ../extensions/message-tags
