---
title: IRCv3.3 Message Tags
layout: spec
work-in-progress: true
updates:
  - message-tags-3.2
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
the `draft/message-tags-0.3` capability name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

This specification extends the [IRCv3.2 Message Tags](./message-tags-3.2.html) specification in a number of ways:

* Adds a new capability to indicate base support for message tags
* Defines a prefix for expressing client-only tags
* Defines a new command for tag-only messages
* Defines a new encoding for sending message tags as JSON
* Increases the size limit for tags

## Motivation

Previously, clients were required to negotiate a capability with servers for each
supported tag. This made tags inappropriate for client-only features. By adding a
new base capability, this specification allows clients to indicate support for
receiving any well-formed tag, whether or not it is recognised or used. This also
frees servers from having to filter individual message tags for each client
response.

The client-only tag prefix allows servers to safely relay untrusted client tags,
keeping them distinct from server-initiated tags that carry verified meaning.

To allow for tagged data to be sent to channels and users without any accompanying
message text, a new command for tag-only messages is needed.

To allow for tag values containing complex typed data such as lists and associative arrays, new encoding rules are required. JSON was chosen as an existing, widely-adopted format with established encoding rules.

With the scope of tags expanded for use as general purpose message metadata, the
number and size of tags attached to a message will potentially increase. As a
result, an increased limit is implied by negotiating the new capability.

## Architecture

### Capabilities

This specification adds the `draft/message-tags-0.3` capability. Clients requesting
this capability indicate that they are capable of parsing all well-formed tags,
even if they don't handle them specifically.

Servers advertising this capability indicate that they are capable of parsing
any tag received from a client, with or without the client-only prefix.

### Tags

Client-only tags are client-initiated tags that servers MUST attach as-is
to any relevant message relayed to other clients. A client-only tag is prefixed
with a plus sign (`+`) and otherwise conforms to the format specified in
[IRCv3.2 tags](./message-tags-3.2.html).

Client-only tags MUST be relayed on `PRIVMSG` and `NOTICE` messages, and MAY be relayed on other messages.

Any server-initiated tags attached to messages MUST be included before client-only
tags to prevent them from being pushed outside of the byte limit.

The expected client behaviour of individual client-only tags SHOULD be defined
in separate specifications, in the same way as server-initiated tags.

This means client-only tags that aren't specified in the IRCv3 extension registry MUST
use a vendor prefix and SHOULD be submitted to the IRCv3 working group for consideration
if they are appropriate for more widespread adoption.

The updated pseudo-BNF for keys is as follows:

    <key> ::= [ '+' ] [ <vendor> '/' ] <sequence of letters, digits, hyphens ('-')>

Individual tag keys MUST only be used a maximum of once per message. Clients
receiving messages with more than one occurrence of a tag key SHOULD discard all
but the final occurrence.

### The `TAGMSG` tag-only message

       Command: TAGMSG
    Parameters: <msgtarget>

A new message command `TAGMSG` is defined for sending messages with tags but no text content.
This message MUST be delivered to targets in the same way as `PRIVMSG` and `NOTICE`
messages. This means for example, honouring channel membership, modes,
[`echo-message`](../extensions/echo-message-3.2.html),
[`STATUSMSG`](https://tools.ietf.org/html/draft-hardy-irc-isupport-00#section-4.18)
prefixes, etc.

Servers MAY apply moderation to this command using existing or newly specified modes or configuration.

Servers MUST NOT deliver `TAGMSG` to clients that haven't negotiated the message tags capability.

Servers SHOULD reject any `TAGMSG` command sent without tags. In this case, they MUST use the `ERR_NEEDMOREPARAMS` (`461`) error numeric.

See [`PRIVMSG` in RFC2812](https://tools.ietf.org/html/rfc2812#section-3.3.1) for more details on replies and examples.

Clients that receive a `TAGMSG` command MUST NOT display them in the message history by default. Display guidelines are defined in the specifications of tags attached to the message.

### JSON encoding format

To send tags as JSON, clients and servers MUST encode tag data as an *object* whos top-level keys are the tag keys, encoded as *strings*

Tag keys without a value MUST be represented in JSON as keys with a `null` value.

Tag values MUST be encoded as JSON *string* types, except where defined in specific tag specifications.

All extraneous whitespace MUST be removed from encoded JSON, and the entire encoded JSON data MUST be escaped according to existing tag value escaping rules.

Fully encoded and escaped JSON tag data is transmitted instead of `<tags>` in the originally defined pseudo-BNF.

To detect JSON-encoded tag data, clients and servers MUST check for `'{'` (0x7B) as the first character of tag data, after the leading `'@'` (0x40) character. The JSON data ends on the first space `' '` (0x20) character.

If JSON tag data fails to decode correctly, clients and servers MUST treat the message as having no tags.

### Size limit

The size limit for message tags is increased from 512 to 4607 bytes, including the leading `'@'` (0x40) and trailing space `' '` (0x20) characters. The size limit for the rest of the message is unchanged.

This limit is separated between server-initiated and client-initiated tags. This prevents servers from overflowing the overall limit by adding tags to a client message sent within the allowed limit.

In the following description, **tag data** describes the bytes between the leading and trailing tag separators mentioned above.

Clients MUST NOT send messages with tag data exceeding 4094 bytes, this includes tags with or without the client-only prefix.

Servers MUST NOT add tag data exceeding 510 bytes to messages.

    <server_max>    (512)  :: '@' <tag_data  510> ' '
    <client_max>   (4096)  :: '@' <tag_data 4094> ' '
    <combined_max> (4607)  :: '@' <tag_data  510> ';' <tag_data 4094> ' '

Servers MUST reply with the `ERR_INPUTTOOLONG` (`417`) error numeric if a client sends a message with more tag data than the allowed limit. Servers MUST NOT truncate tags but MUST always reject lines that exceed this limit.

    417    ERR_INPUTTOOLONG
          ":Input line was too long"

## Security considerations

Client-only tags should be treated as untrusted data. They can contain any value
and are not validated by servers in any way. The server MAY unescape tag values
after receiving data and escape those values before sending them out.

Some specifications may involve servers accepting client-initiated tags without
the client-only prefix, they SHOULD define a validation process to be performed
by the server.

Tags without the client-only prefix that are not specified to be relayed by the
server or that fail validation MUST be removed by the server before being sent
with any response to another client.

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

A message with the same tags sent with both the normal and JSON encodings for comparison:

    @tag=value;+client-tag=value\swith\sspaces PRIVMSG #channel :Message
    @{"tag":"value","+client-tag":"value\swith\sspaces"} PRIVMSG #channel :Message

---

A message with complex typed data that can ONLY be sent with the JSON encoding:

    @{"+complex-tag":[{"key1":[1,2,3]},{"key\s2":["a","b","c"]}]} PRIVMSG #channel :Message

---

In this example

* The user `nick!user@example.com` shares a URL in a channel (without tags)
* The bot `url_bot!bot@example.com` responds with the URL title in the message body and the favicon URL included as the value of the `+icon` client-only tag:

The server sends these messages:

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

A `TAGMSG` sent by a client with the `+example-client-tag` client-only tag:

    C: @+example-client-tag=example-value TAGMSG #channel

---

A `TAGMSG` sent by a client to channel ops with a client-only tag and `STATUSMSG` prefix:

    C: @+example-client-tag=example-value TAGMSG @#channel

---

A `TAGMSG` sent to a channel with [`labeled-response`](../extensions/labeled-response.html) and client-only tags. The message is returned by the server with a [`msgid`](../extensions/message-ids.html) tag via labeled `echo-message` to the originating client and as normal to other clients in the channel.

    C1-S: @label=123;+example-client-tag=example-value TAGMSG #channel
    S-C1: @label=123;msgid=abc;+example-client-tag=example-value :nick!user@example.com TAGMSG #channel
    S-C2: @msgid=abc;+example-client-tag=example-value :nick!user@example.com TAGMSG #channel

---

A `TAGMSG` sent by a client without any tags and rejected by the server with an `ERR_NEEDMOREPARAMS` (`461`) error numeric.

    C: TAGMSG #channel
    S: :server.example.com 461 nick TAGMSG :Not enough parameters

---

A `TAGMSG` sent by a client with tags that exceed the size limit and rejected by the server with an `ERR_INPUTTOOLONG` (`417`) error numeric. `[...]` is used to represent tags omitted for readability.

    C: @+tag1;+tag2;+tag[...];+tag5000 TAGMSG #channel
    S: :server.example.com 417 nick :Input line was too long
