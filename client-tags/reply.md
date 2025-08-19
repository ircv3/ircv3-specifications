---
title: "`reply` client tag"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "James Wheare"
    period: 2016
    email: "james@irccloud.com"
---

## Introduction

This specification defines a client-only message tag to indicate replies to other messages

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the [`message-tags`](../extensions/message-tags.html) capability with the server. Additionally, this tag relies on messages being sent with the [`msgid`](../extensions/message-ids.html) tag. Clients SHOULD negotiate the [`echo-message`](../extensions/echo-message.html) capability in order to receive message IDs for their own messages, and therefore understand any replies.

### Format

The reply tag is sent by a client with the client-only prefix `+` and its value references the server provided ID of another message:

    +reply=<msgid>

## Client implementation considerations

This section is non-normative

When displaying replies, take care with the chronology of messages. Clients might choose to display a reply message adjacent to its parent, to make a "thread" apparent. This might work well when the messages are close together in the message history, but could cause problems if there have been several intervening messages in the mean time. People following the conversation may not see a reply if it's hoisted too far into the backlog.

In this situation, it might make more sense to leave the reply in place chronologically, but provide a way to jump to the original message instead.

## Examples

In this example, a `PRIVMSG` is sent to a channel with an ID provided by the server. A client sends a reply to this message and the server sends an echo-message back to the client.

    S: @msgid=123 :nick!user@host PRIVMSG #channel :Hello!
    C: @+reply=123 PRIVMSG #channel :Hello to you!
    S: @msgid=456;+reply=123 :nick2!user2@host2 PRIVMSG #channel :Hello to you!
