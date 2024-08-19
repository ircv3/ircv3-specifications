---
title: "`notify` client tag"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Sadie Powell"
    period: "2024"
    email: "sadie@witchery.services"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `+notify` tag name. Instead, implementations SHOULD use the `+draft/notify` tag name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification defines a client-only message tag to indicate to the receiving client how they should notify the user.

## Motivation

Sometimes when you send a message to a user you don't want to bother them immediately. Unfortunately, it is not currently possible to control whether a user will be notified by a message. This specification defines a method for controlling notifications in compatible clients.

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the [`message-tags`](../extensions/message-tags.html) capability with the server. Additionally, this tag MUST be used in conjunction with the [`+draft/reply`](./reply.html) client tag.

### Format

The reply tag is sent by a client with the client-only prefix `+`. The format of this tag is:

    +draft/notify=<value>

Possible values are:

- `never` &mdash; Do not notify the user.
- `active` &mdash; Only notify the user if they are actively using their computer.
- `online` &mdash; Only notify the user if they are attached to their client. For clients which do not support detaching this SHOULD be treated the same as `active`.

Any other values are undefined and SHOULD be treated as if the tag was not present.

This tag SHOULD only be attached to a `NOTICE`, `PRIVMSG`, or `TAGMSG` message.

## Examples

An example of a notify-never message sent to a channel. Without the notify tag it might notify *val*.

    C: @+draft/notify=never PRIVMSG #ircdocs :I'm not sure, try asking val when they're around?

An example of a notify-active message sent to a user. Without the notify tag it might notify *Adam* when their computer is idle.

    C: @+draft/notify=active PRIVMSG Adam :I've done the release

An example of a notify-online message sent to a user. Without the notify tag it might notify *Sadie* when they are not attached to their client.

    C: @+draft/notify=never PRIVMSG Sadie :Are you busy? No worries if not.

## Implementation Considerations

It is up to client implementers to determine if to determine if their user has been notified by a message.

<!-- TODO: More considerations needed here -->
