---
title: Persist tag
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Shivaram Lingamneni"
    period: 2020
    email: "slingamn@cs.stanford.edu"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `persist` tag name. Instead, implementations SHOULD use the `draft/persist` tag name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification defines a message tag to recommend that a TAGMSG be stored by the server.

## Motivation

A variety of proposed extensions involve clients sending TAGMSG that have only temporary relevance. Some examples include:

* [Typing notifications](https://ircv3.net/specs/client-tags/typing)
* Autocompletion requests for IRC bots
* Read receipts for messages

These messages have some salient common characteristics:

1. They have limited or negligible relevance outside the immediate temporal context in which they were sent
1. They have limited or negligible impact on the course of the "conversation" among its human participants (unlike, say, reactions)
1. Because they are machine-readable, they may be sent at a higher rate or volume than conventional PRIVMSG or NOTICE carrying human-readable text

It is desirable to be able to advise the server not to store them and replay them to offline clients --- either because they would consume server resources unnecessarily, or because replaying them would actually risk degrading the client user experience.

In contrast, some extensions involve TAGMSG that have more enduring relevance, in particular, [reactions](./react). If a server offers history storage and replay functionality, then it should store and replay these messages.

The question remains whether storage of TAGMSG should be "opt-in" or "opt-out". However, "opt-in" seems more resilient to client implementation issues and is a better fit for the preponderance of client-only tags that have been proposed thus far. Accordingly, the persist tag designates TAGMSG for opt-in storage, with the intent that servers will store only those TAGMSG that carry the persist tag.

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the [`message-tags`](../extensions/message-tags.html) capability with the server.

### Format

The persist tag is sent by a client, but without the client-only prefix `+`. It has no value:

    draft/persist

## Server implementation considerations

Servers SHOULD NOT store TAGMSG that do not contain the `draft/persist` tag. If a server implements history storage and replay, it SHOULD store TAGMSG that contain the `draft/persist` tag. Servers SHOULD ignore the presence of `draft/persist` on all other message types.

Since the specification of `+draft/react` precedes this specification, servers MAY wish to store TAGMSG that carry `+draft/react` but not `draft/persist`.

Note that although `draft/persist` is sent by the client, it is not a "client-only tag" in the sense of the message tags specification and SHOULD NOT be relayed to other clients.

## Client implementation considerations

Clients SHOULD attach `draft/persist` to any TAGMSG that they believe should be stored for replay. Future client tag specifications should recommend the use of `draft/persist` when relevant.

## Examples

This is an example of a reaction with `draft/persist` attached:

    C: @+draft/reply=123;+draft/react=lol;draft/persist TAGMSG #channel

## Other Considerations

This section is non-normative.

The `draft/persist` tag is not intended as a solution to abusive client behavior. Servers may wish to implement other safeguards against client attempts to exhaust server-side resources.
