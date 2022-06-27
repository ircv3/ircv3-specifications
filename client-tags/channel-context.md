---
title: "`channel-context` client tag"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "delthas"
    period: 2022
    email: "delthas@dille.cc"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `+channel-context` tag name. Instead, implementations SHOULD use the
`+draft/channel-context` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification defines a client-only message tag to indicate the channel a private PRIVMSG or NOTICE should be displayed in.

## Motivation

Responding to PRIVMSG messages in channels with NOTICE messages in private is a widely-used IRC practice for bots, in order to prevent flooding a channel with messages only useful for a single user. Clients have developed heuristics for matching incoming private NOTICE to their channel context, but these heuristics are error-prone and difficult to implement for past messages with CHATHISTORY.

This tag enables clients to systematically know in which channel a private PRIVMSG or NOTICE should be displayed, because the sender can now communicate that information in the message.

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the [`message-tags`](../extensions/message-tags.html) capability with the server.

### Format

The channel-context tag is sent by a client with the client-only prefix `+`. The value MUST be a valid channel name.

    +draft/channel-context=<channel>

This tag MUST be attached to PRIVMSG and NOTICE messages to users only. Clients MUST ignore this tag if it is not attached to a private PRIVMSG or NOTICE message or if its value is not a valid channel name.

## Client implementation considerations

This section is non-normative.

When clients receive a private PRIVMSG or NOTICE message from a user with this tag, they can display that message in the buffer specified in the tag, as if it was a channel message.

To prevent abuse, client implementations might allow users to hide these messages when the sender does not belong to the specified channel.

## Examples

In this example, user `me` requests documentation from bots in channel `#chan`, then gets the response in private, but in the context of that channel, using `channel-context`.

    C: PRIVMSG #chan :!help
    S: @+draft/channel-context=#chan :bot!bot@bot NOTICE me :I'm helping.
