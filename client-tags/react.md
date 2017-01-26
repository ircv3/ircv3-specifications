---
title: React client tag
layout: spec
work-in-progress: true
copyrights:
  -
    name: "James Wheare"
    period: 2016
    email: "james@irccloud.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `+react` tag name. Instead, implementations SHOULD use the
`+draft/react` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification defines a client-only message tag to indicate reactions to other messages

## Motivation

This tag provides a means of communicating with context-sensitive, potentially non-textual reactions. It allows chat participants to respond to each other with text, symbols, emoticons, or emoji that don't necessarily appear as full messages, but instead as lightweight annotations, displayed adjacent to a parent message.

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the [`draft/message-tags`](../core/message-tags-3.3.html) capability with the server. Additionally, this tag MUST be used in conjunction with the [`+draft/reply`](./reply.html) client tag.

### Format

The react tag is sent by a client with the client-only prefix `+`. The value has no restrictions.

    +draft/react=<reaction>

This tag MAY be attached to an empty message.

## Client implementation considerations

This section is non-normative

When reactions are sent as empty messages, they may not be visible to clients that don't support them. This may be the desired outcome, since they're intended as lightweight responses whose meaning might be lost when detached from their parent context. They may also be numerous and take up more room in the message history than intended by the sender.

However, reactions might also create their own context, sparking further conversation, so it might make sense to display them as a full message body.

Clients might wish to enable a preference to allow their users to choose wheter they want to send reactions as tag-only or duplicated in the message body.

Other reasons for users not to see a reaction might include:

* Server-side filtering or moderation
* Archived message history being viewed in a client without all subsequent replies

### Value restrictions

This specification doesn't define any restrictions on what can be sent as the reaction value. Clients might wish to apply their own restrictions for which values are allowed to be sent or received. For instance, a byte limit may be appropriate, or white/blacklists.

## Examples

In this example, a `PRIVMSG` is sent to a channel with an ID provided by the server. A client sends a reaction reply to this message and the server sends an echo-message back to the client.

    S: @draft/msgid=123 :nick!user@host PRIVMSG #channel :Hello!
    C: @+draft/reply=123;+draft/react=lol :nick2!user2@host2 PRIVMSG #channel
    S: @draft/msgid=456;+draft/reply=123;+draft/react=lol :nick2!user2@host2 PRIVMSG #channel

An example of an emoticon reaction

    C: @+draft/reply=123;+draft/react=:) :nick2!user2@host2 PRIVMSG #channel

An example of an emoji reaction

    C: @+draft/reply=123;+draft/react=👋 :nick2!user2@host2 PRIVMSG #channel

An example of a reaction with an additional message body

    C: @+draft/reply=123;+draft/react=lol :nick2!user2@host2 PRIVMSG #channel :lol
