---
title: IRCv3 Message Tags
layout: spec
redirect_from:
  - /specs/core/message-tags-3.3.html
  - /specs/core/message-tags-3.2.html
copyrights:
  -
    name: "Alexey Sokolov"
    period: 2012-2014
    email: "alexey-irc@asokolov.org"
  -
    name: "Stéphan Kochen"
    period: "2012"
    email: "stephan@kochen.nl"
  -
    name: "Kyle Fuller"
    period: "2012"
    email: "inbox@kylefuller.co.uk"
  -
    name: "Kiyoshi Aman"
    period: "2016"
    email: "kiyoshi.aman@gmail.com"
  -
    name: "James Wheare"
    period: "2016-2019"
    email: "james@irccloud.com"
---

## Introduction

Message tags are a mechanism for adding additional metadata on a per-message basis. This is achieved via an extension to the protocol message format, enabled via capability negotiation.

Tags are simple keys that MAY have optional string data as values.

Tagged messages can be sent by both servers and clients. The usage of individual tags are specified in their own documents.

## Format

The message pseudo-BNF, as defined in [RFC 1459, section 2.3.1][rfc1459] is extended as follows:

    <message>       ::= ['@' <tags> <SPACE>] [':' <prefix> <SPACE> ] <command> <params> <crlf>
    <tags>          ::= <tag> [';' <tag>]*
    <tag>           ::= <key> ['=' <escaped_value>]
    <key>           ::= [ <client_prefix> ] [ <vendor> '/' ] <key_name>
    <client_prefix> ::= '+'
    <key_name>      ::= <sequence of letters, digits, hyphens ('-')>
    <escaped_value> ::= <sequence of any characters except NUL, CR, LF, semicolon (`;`) and SPACE>
    <vendor>        ::= <host>

The ordering of tags is not meaningful.

Individual tag keys MUST only be used a maximum of once per message. Clients
receiving messages with more than one occurrence of a tag key SHOULD discard all
but the final occurrence.

## Escaping values

The mapping between characters in tag values and their representation in `<escaped value>` is defined as follows:

| Character       | Sequence in `<escaped value>` |
|-----------------|-------------------------------|
| `;` (semicolon) | `\:` (backslash and colon)    |
| `SPACE`         | `\s`                          |
| `\`             | `\\`                          |
| `CR`            | `\r`                          |
| `LF`            | `\n`                          |
| all others      | the character itself          |

This escape format is space efficient, and ensures that message parts can easily be split on spaces and semi-colons before further parsing.

If a lone `\` exists at the end of an escaped value (with no escape character following it), then there
SHOULD be no output character. For example, the escaped value `test\` should unescape to `test`.

## Capabilities

Tags are enabled via capability negotiation. Clients and servers that negotiate a capability that uses tags MUST support the full tag specification.

The following capabilities implicitly depend on the tag format:

* `account-tag`
* `server-time`
* `batch`

But there is also a generic capability for negotiating tag usage.

* `message-tags`

### `message-tags` capability

The `message-tags` capability allows clients to indicate support for
receiving any well-formed tag, whether or not it is recognised or used. It also frees servers from having to filter individual message tags for each client response.

This capability enables the use of client-only tags and the `TAGMSG` command, described below.

Clients requesting this capability indicate that they are capable of parsing all well-formed tags, even if they don't handle them specifically.

Servers advertising this capability indicate that they are capable of parsing
any tag received from a client, with or without the client-only prefix.

### Client-only tags

Client-only tags are tags normally sent by clients that servers MUST attach as-is
to any relevant message relayed to other clients. A client-only tag is prefixed
with a plus sign (`+`). Servers MAY send client-only tags that weren't provided explicitly by the client.

The client-only tag prefix allows servers to safely relay untrusted client tags,
keeping them distinct from unprefixed tags that carry verified meaning.

Client-only tags MUST be relayed on `PRIVMSG` and `NOTICE` messages, and MAY be relayed on other messages.

Any server-sent tags attached to messages MUST be included before client-only
tags to prevent them from being pushed outside of the byte limit.

The expected client behaviour of individual client-only tags are defined
in separate specifications.

This means client-only tags that aren't specified in the IRCv3 extension registry MUST
use a vendor prefix and SHOULD be submitted to the IRCv3 working group for consideration
if they are appropriate for more widespread adoption. See [Rules for naming message tags](#rules-for-naming-message-tags).

### The `TAGMSG` tag-only message

       Command: TAGMSG
    Parameters: <msgtarget>

A new message command `TAGMSG` is defined for sending messages with tags but no text content.
This message MUST be delivered to targets in the same way as `PRIVMSG` and `NOTICE`
messages. This means for example, honouring channel membership, modes,
[`echo-message`](../extensions/echo-message-3.2.html),
[`STATUSMSG`][statusmsg]
prefixes, etc.

Servers MAY apply moderation to this command using existing or newly specified modes or configuration.

Servers MUST NOT deliver `TAGMSG` to clients that haven't negotiated the message tags capability.

See [`PRIVMSG` in RFC2812][privmsg] for more details on replies and examples.

Clients that receive a `TAGMSG` command MUST NOT display them in the message history by default. Display guidelines are defined in the specifications of tags attached to the message.

## Rules for naming message tags

Full tag names, including any vendor prefixes MUST be treated as an opaque identifier.

There are two tag namespaces:

### Vendor-Specific

Names which contain a slash character (`/`) designate a vendor-specific tag namespace.
These names are prefixed by a valid DNS domain name.

For example: `znc.in/server-time`.

In cases where the prefix contains non-ASCII characters, punycode MUST be used,
e.g. `xn--e1afmkfd.org/foo`.

Vendor-Specific tags should be submitted to the IRCv3 working group for consideration.

### Standardized

Reserved names for which a corresponding document exists in the [IRCv3 Extension Registry][registry].

The IRCv3 Working Group reserves the right to reuse names which have not been submitted
to the registry. If you do not wish to submit your tag then you MUST use a vendor-specific
name (see above).

## Size limit

The size limit for message tags is 4607 bytes, including the leading `'@'` (0x40) and trailing space `' '` (0x20) characters. The size limit for the rest of the message remains unchanged at 512 bytes.

This limit is separated between tags added by the server and tags sent by the client. This prevents servers from overflowing the overall limit by adding tags to a client message sent within the allowed limit.

In the following description, **tag data** describes the bytes between the leading and trailing tag separators mentioned above.

Clients MUST NOT send messages with tag data exceeding 4094 bytes, this includes tags with or without the client-only prefix.

Servers MUST NOT add tag data exceeding 510 bytes to messages.

    <server_max>    (512)  :: '@' <tag_data  510> ' '
    <client_max>   (4096)  :: '@' <tag_data 4094> ' '
    <combined_max> (4607)  :: '@' <tag_data  510> ';' <tag_data 4094> ' '

Servers MUST reply with the `ERR_INPUTTOOLONG` (`417`) error numeric if a client sends a message with more tag data than the allowed limit. Servers MUST NOT truncate tags and MUST always reject lines that exceed this limit.

    417    ERR_INPUTTOOLONG
          ":Input line was too long"

## Security considerations

Client-only tags should be treated as untrusted data. They can contain any value
and are not validated by servers in any way. The server MAY unescape tag values
after receiving data and escape those values before sending them out.

Tags without the client-only prefix MUST be removed by the server before being relayed
with any message to another client.

Some specifications may involve servers accepting client-sent tags without
the client-only prefix. These specifications MUST define a process to be performed by the server
on these tags prior to their removal.

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

A message with 0 tags:

    :nick!ident@host.com PRIVMSG me :Hello

---

A message sent by the server with 3 tags:

    S: @aaa=bbb;ccc;example.com/ddd=eee :nick!ident@host.com PRIVMSG me :Hello

* standardized tag `aaa` with value `bbb`

* standardized tag `ccc` with no value

* tag `ddd` specific to software of `example.com` with value `eee`

---

A message sent by a client with the `example-tag` tag:

    C: @example-tag=example-value PRIVMSG #channel :Message

---

A message sent by a client with the `+example-client-tag` client-only tag:

    C: @+example-client-tag=example-value PRIVMSG #channel :Message

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

In this example, plus signs, colons, equals signs and commas are transmitted raw in tag values; while semicolons, spaces and backslashes are escaped.

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

A `TAGMSG` sent by a client with tags that exceed the size limit and rejected by the server with an `ERR_INPUTTOOLONG` (`417`) error numeric. `[...]` is used to represent tags omitted for readability.

    C: @+tag1;+tag2;+tag[...];+tag5000 TAGMSG #channel
    S: :server.example.com 417 nick :Input line was too long

---

A `TAGMSG` sent by a client with an un-prefixed tag that has no specified behaviour. The server removes the tag before relaying the message

    C: @unknown-tag TAGMSG #channel
    S: :nick!user@example.com TAGMSG #channel

## Errata

Previous versions of this spec did not specify that the full tag name MUST be parsed as
an opaque identifier. This was added to improve client resiliency.

Previous versions of this spec did not specify how to handle trailing backslashes with
no escape character. This was added to help consistency across implementations.

[rfc1459]: http://tools.ietf.org/html/rfc1459#section-2.3.1
[privmsg]: https://tools.ietf.org/html/rfc2812#section-3.3.1
[statusmsg]: https://tools.ietf.org/html/draft-hardy-irc-isupport-00#section-4.18
[registry]: https://ircv3.net/registry.html#tags