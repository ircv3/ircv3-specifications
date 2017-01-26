---
title: IRCv3.3 Message Tags
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Kiyoshi Aman"
    period: 2016
    email: "kiyoshi.aman@gmail.com"
  -
    name: "James Wheare"
    period: 2016
    email: "james@irccloud.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `message-tags` capability name. Instead, implementations SHOULD use
the `draft/message-tags-0.2` capability name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

This specification adds a new capability for general message tags support and
a prefix for expressing client-only tags. It also defines a new event for
tag-only messages, and increases the byte limit for tags.

## Motivation

Previously, clients were required to negotiate a capability with servers for each
supported tag. This made tags inappropriate for client-only features. By adding a
new base capability, this specification allows clients to indicate support for
receiving any well-formed tag, whether or not it is recognised or used. This also
frees servers from having to filter message tags for each individual client
response.

The client-only tag prefix allows servers to safely relay untrusted client tags,
keeping them distinct from server-initiated tags that carry verified meaning.

To allow for tagged data to be sent to channels and users without any accompanying
message text, a new event for tag-only messages is needed.

With the scope of tags expanded for use as general purpose message metadata, the
number and size of tags attached to a message will potentially increase. As a
result, an increased limit is implied by negotiating the new capability.

## Architecture

### Capabilities

This specification adds the `draft/message-tags-0.2` capability. Clients requesting
this capability indicate that they are capable of parsing all well-formed tags,
even if they don't handle them specifically.

Servers advertising this capability indicate that they are capable of parsing
any tag received from a client, with or without the client-only prefix.

### Tags

Client-only tags are client-initiated tags that servers MUST attach as-is
to any relevant event relayed to other clients. A client-only tag is prefixed
with a plus sign (`+`) and otherwise conforms to the format specified in
[IRCv3.2 tags](./message-tags-3.2.html).

Client-only tags MUST be relayed on `PRIVMSG` and `NOTICE` events, and
MAY be relayed on other events.

Any server-initiated tags attached to messages MUST be included before client-only
tags to prevent them from being pushed outside of the 512 byte tag limit.

The expected client behaviour of individual client-only tags SHOULD be defined
in separate specifications, in the same way as server-initiated tags.

This means client-only tags that aren't specified in the IRCv3 extension registry MUST
use a vendor prefix and SHOULD be submitted to the IRCv3 working group for consideration
if they are appropriate for more widespread adoption.

The updated pseudo-BNF for keys is as follows:

    <key> ::= [ '+' ] [ <vendor> '/' ] <sequence of letters, digits, hyphens (`-`)>

Individual tag keys MUST only be used a maximum of once per message. Clients
receiving messages with more than one occurrence of a tag key SHOULD discard all
but the final occurrence.

### The `TAGMSG` tag-only event

A new event `TAGMSG` is defined for sending messages with tags but no text content.
This event MUST be delivered to targets in the same way as `PRIVMSG` and `NOTICE`
events. This means for example, honouring channel membership, modes,
[`echo-message`](../extensions/echo-message-3.2.html),
[`STATUSMSG`](https://tools.ietf.org/html/draft-hardy-irc-isupport-00#section-4.18)
prefixes, etc.

Servers MAY apply moderation to this event using existing or newly specified modes or
configuration.

This event MUST NOT be delivered to clients that haven't negotiated the message tags
capability.

If this event is sent without any tags, servers SHOULD reject it. In this case, they
MUST use the standard `ERR_NEEDMOREPARAMS` (`461`) error numeric.

See [`PRIVMSG` in RFC1459](https://tools.ietf.org/html/rfc1459#section-4.4.1) for more details on replies and examples.

Clients that receive the `TAGMSG` event MUST NOT display them in the message history
except according to the specifications of the attached tags.

The pseudo-BNF for this event is as follows:

    <message> ::= '@' <tags> <SPACE> [':' <prefix> <SPACE> ] 'TAGMSG' <SPACE> <target> <crlf>

### Size limit

The size limit for message tags is increased from 512 to 4096 bytes, including the leading
`@` and trailing space characters, leaving 4094 bytes for tags themselves. The size limit
for the rest of the message is unchanged.

## Security considerations

Client-only tags should be treated as untrusted data. They can contain any value
and are not validated by servers in any way.

Some specifications may involve servers accepting client-initiated tags without the client-only prefix. Such tag values SHOULD be validated by the server before being sent with any response to a client, unless they are specifically specified to be untrusted data.

## Client implementation considerations

There is no guarantee that other clients will support or display the client-only
tags they receive, so these tags should only be used to enhance messages with non-critical
information. Messages should still make sense to clients with no support for tags.

## Moderation considerations

This section is non-normative.

Moderation tools available to channel and network operators typically operate on message
text and sender information only. To avoid abusive content being sent in client-only tags,
servers, services and management bot implementations may wish to enable moderation features
that act on tag content as well.

## Examples

This section is non-normative. The tags used in these examples may or may not have a specified meaning elsewhere.

---

A message sent by a client with the `example-tag` tag:

    C: @example-tag=example-value PRIVMSG #channel :Message

---

A message sent by a client with the `+example-client-tag` client-only tag:

    C: @+example-client-tag=example-value PRIVMSG #channel :Message

---

Server responses for:

* The user `nick` sharing a URL in a channel (without tags)
* The bot `url_bot` responding with the URL title in the message body and the favicon URL included as the value of the `+icon` client-only tag:


    S: :nick!user@example.com PRIVMSG #channel :https://example.com/a-news-story
    S: @+icon=https://example.com/favicon.png :url_bot!bot@example.com PRIVMSG #channel :Example.com: A News Story

---

An example of a vendor-prefixed client-only tag:

    C: @+example.com/foo=bar :irc.example.com NOTICE #channel :A vendor-prefixed client-only tagged message

---

A client-only tag `+example` with a value containing valid raw and escaped characters:

* Raw value: `raw+:=,escaped; \`
* Escaped value: `raw+:=,escaped\:\s\\`

In this example, plus signs, colons, equals signs and commas are transmitted raw in tag values; while semicolons, spaces and backslashes are escaped. [Escaping rules](./message-tags-3.2.html#escaping-values) are unchanged from IRCv3.2 tags.

    C: @+example=raw+:=,escaped\:\s\\ :irc.example.com NOTICE #channel :Message

---

A TAGMSG event sent by a client with the `+example-client-tag` client-only tag:

    C: @+example-client-tag=example-value TAGMSG #channel

---

A TAGMSG event sent by a client to channel ops with a client-only tag and `STATUSMSG` prefix:

    C: @+example-client-tag=example-value TAGMSG @#channel

---

A TAGMSG event sent to a channel with [`labeled-response`](../extensions/labeled-response.html) and client-only tags. The event is returned by the server with a [`msgid`](../extensions/message-ids.html) tag via labeled `echo-message` to the originating client and as normal to other clients in the channel.

    C1-S: @label=123;+example-client-tag=example-value TAGMSG #channel
    S-C1: @label=123;msgid=abc;+example-client-tag=example-value :nick!user@example.com TAGMSG #channel
    S-C2: @msgid=abc;+example-client-tag=example-value :nick!user@example.com TAGMSG #channel

---

A TAGMSG event sent by a client without any tags and rejected by the server with an `ERR_NEEDMOREPARAMS` (`461`) error numeric.

    C: TAGMSG #channel
    S: :server.example.com 461 nick TAGMSG :Not enough parameters
