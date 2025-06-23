---
title: "`react` client tag"
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
unprefixed `+react` or `+unreact` tag names. Instead, implementations SHOULD use the
`+draft/react` and `+draft/unreact` tag names to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use unprefixed tag names.

## Introduction

This specification defines a client-only message tag to indicate reactions and unreactions to other messages

## Motivation

This tag provides a means of communicating with context-sensitive, potentially non-textual reactions. It allows chat participants to respond to each other with text, symbols, emoticons, or emoji that don't necessarily appear as full messages, but instead as lightweight annotations, displayed adjacent to a parent message.

While most IRC users do not expect messages to be removable, reactions are meant to be sent quickly, changed based on opinion and removed if sent on accident. For this reason, most implementations of emoji reactions allow an unreaction feature.

One may wonder why an unreaction tag is needed when the [`draft/message-redaction`](../extensions/message-redaction.md.html) also allows for message removal. However, these two methods of message removal serve different purposes. Redaction is a server-moderated process to hide previous client actions, e.g., such that they do not show up in history. This capability is usually only given to users authorized to hide the action, such as a logged-in author or operators. 

On the contrary, unreaction is its own action designed to change a previous choice (e.g., in response to a “voting” message), or remedy a misclick. If sent to clients as a text message, previous reactions need not be removed on an unreact message.

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the [`message-tags`](../extensions/message-tags.html) capability with the server. Additionally, this tag MUST be used in conjunction with the [`+draft/reply`](./reply.html) client tag.

### Format

The react tag is sent by a client with the client-only prefix `+`. The value has no restrictions.

    +draft/react=<reaction>

The unreact tag is sent by a client with the client-only prefix `+`. The value has no restrictions.

    +draft/unreact=<reaction>

These tags MAY be attached to an empty message.

## Client implementation considerations

This section is non-normative

When reactions are sent as empty messages, they may not be visible to clients that don't support them. This may be the desired outcome, since they're intended as lightweight responses whose meaning might be lost when detached from their parent context. They may also be numerous and take up more room in the message history than intended by the sender.

However, reactions might also create their own context, sparking further conversation, so it might make sense to display them as a full message body.

Clients might wish to enable a preference to allow their users to choose whether they want to send reactions as tag-only or duplicated in the message body.

Other reasons for users not to see a reaction might include:

* Server-side filtering or moderation
* Archived message history being viewed in a client without all subsequent replies

### Value restrictions

This specification doesn't define any restrictions on what can be sent as the reaction value. Clients might wish to apply their own restrictions for which values are allowed to be sent or received. For instance, a byte limit may be appropriate, or white/blacklists.

## Examples

In this example, a `TAGMSG` is sent to a channel with an ID provided by the server. A client sends a reaction reply to this message and the server sends an echo-message back to the client.

    S: @msgid=123 :nick!user@host PRIVMSG #channel :Hello!
    C: @+draft/reply=123;+draft/react=lol TAGMSG #channel
    S: @msgid=456;+draft/reply=123;+draft/react=lol :nick2!user2@host2 TAGMSG #channel

An example of an emoticon reaction

    C: @+draft/reply=123;+draft/react=:) TAGMSG #channel

An example of an emoji reaction

    C: @+draft/reply=123;+draft/react=👋 TAGMSG #channel

An example of a reaction sent as a `PRIVMSG` with an additional message body

    C: @+draft/reply=123;+draft/react=lol PRIVMSG #channel :lol

An example of an reaction and unreaction sent as a `TAGMSG`s

    S: @msgid=123 :nick!user@host PRIVMSG #football :They won!
    C: @+draft/reply=123;+draft/react=🇦🇷 TAGMSG #football
    S: @msgid=124 :nick!user@host PRIVMSG #football :Actually it was Germany...
    C: @+draft/reply=123;+draft/unreact=🇦🇷 TAGMSG #football
    C: @+draft/reply=123;+draft/react=🇩🇪 TAGMSG #football
