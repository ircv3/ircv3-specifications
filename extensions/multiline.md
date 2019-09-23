---
title: Multiline messages
layout: spec
work-in-progress: true
copyrights:
  -
    name: "James Wheare"
    period: "2019"
    email: "james@irccloud.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `multiline-concat` tag or `multiline` CAP/batch names. Instead, implementations SHOULD use the `draft/multiline-concat` tag, `draft/multiline` CAP and `draft/multiline` batch names to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification adds a new batch type and tag sent by clients and servers to send messages that can exceed the usual byte length limit and that can contain line breaks.

## Motivation

IRC messages have been traditionally limited by the line-based nature of the protocol and the line length limit of 512. To work around these limitations, client implementations split longer messages into multiple lines, and users need to rely on snippet hosting services to send multiple lines without flooding the chat or having their messages interrupted by other users.

Mulitline messages allow for a more coherent experience that avoids the disconnected messages that result from these workarounds.

## Architecture

### Dependencies

This specification uses the [batch][] capability which MUST be negotiated at the same time. It also uses the [message tags][] and [standard replies][] frameworks.

### Capabilities

This specification adds the `draft/multiline` capability.

Implementations requesting this capability indicate that they are capable of handling the batch type and message tag described below.

The capability has a REQUIRED value: a comma (`,`) (0x2C) separated list of tokens. Each token consists of a key which might have a value attached. If there is a value attached, the value is separated from the key by an equals sign (`=`) (0x3D). That is, `<key>[=<value>][,<key2>[=<value2>][,<keyN>[=<valueN>]]]`. Keys specified in this document MUST only occur at most once.

Clients MUST ignore every token with a key that they don't understand.

The only defined capability key so far is:

* `max-bytes` - This defines the maximum allowed total byte length of multiline batched content *REQUIRED*
* `max-lines` - This defines the maximum allowed number of lines in a multiline batch content *OPTIONAL*

### Tags

This specification adds the `draft/multiline-concat` message tag, which has no value and is used to send message lines that extend beyond the usual byte length limit. Its usage is described below.

### Batch types

This specification adds the `draft/multiline` batch type, and introduces client initiated batches. These use the same syntax as server initiated batches.

In addition to the base batch parameters (reference-tag and type) a multiline batch has one additional parameter, the target recipient.

Multiline batches MUST contain one or more PRIVMSG lines whose target MUST match the batch target.

When receiving a well-formed mulitiline message batch, implementations MUST collect each PRIVMSG message content and wait until the full batch has been received before processing the message. Processing in this context refers to:

* Servers: delivering the batch to the intended recipients
* Clients: displaying the batched message to the user

Messages in a multiline batch MUST be concatenated with a line break unless the `draft/multiline-concat` message tag is sent, in which case the message is directly concatenated with the previous message without any separation.

Clients MUST NOT send messages other than PRIVMSG while a multiline batch is open.

### Fallback

When delivering multiline batches to clients that have not negotiated the multiline capability, servers MUST deliver the component messages without using a multiline BATCH.

Any tags that would have been added to the batch, e.g. message IDs, account tags etc MUST be included on the first message line. Tags MAY also be included on subsequent lines where it makes sense to do so.

### Errors

Servers MUST use `FAIL` messages from the [standard replies][] framework to notify clients of errors with multiline batches. The command is `BATCH` and the following codes are defined:

* `MULTILINE_MAX_BYTES <limit>`: `max-bytes` limit exceeded
* `MULTILINE_MAX_LINES <limit>`: `max-lines` limit exceeded
* `MULTILINE_INVALID_TARGET <batch-target> <provided-target>`: mismatched PRIVMSG target within a batch
* `MULTILINE_INVALID`: any other error

## Interaction with other specifications

### Message tags ([spec][message tags])

Clients MUST only send non-`batch` tags on the opening BATCH command when sending multiline batches.

### Message ids ([spec][message ids])

Servers MUST only include a message ID on the first message of a batch when sending a fallback to non-supporting clients.

### Labeled response ([draft][labeled response]) with echo-message ([spec][echo message])

Servers MUST only include the label tag on the opening BATCH command when replying to a client using `echo-message`.

## Client implementation considerations

This section is non-normative.

### Splitting long lines

When splitting long lines, care is required to ensure messages are displayed appropriately when combined or in the fallback case.

The recommended splitting method is to split long lines between words where possible and leave the space character at the end of the line. This ensures that concatenated lines don't lose the space, while fallback lines don't start with a space.

In order to calculate the maximum allowed space before splitting a line, consider the following line:

```
:nick!~user@host PRIVMSG #channel :hello \n\r
```

The parts of a message that must fit within the 512 byte limit are as follows

| part | syntax | length |
|-|-|-|
| leading | `:` | 1 |
| nick | `nick` | variable |
| nick/mask separator | `!` | 1 |
| user | `~user` | variable |
| user/host separator | `@` | 1 |
| host | `host` | variable |
| command | `PRIVMSG` | 7 |
| channel | `#channel` | variable |
| message separator | ` :` | 2 |
| crlf | `\n\r` | 2 |
| message | `hello ` | variable |

Not all parts may appear in a message, but a safe limit should take them into account and include an extra safety buffer in case of any unexpected syntax added by a server, e.g. a statusmsg prefix.

With a safety buffer of 10, the total message size available before a split is needed can be calculated as follows

```
512 - 14 - 10 - <nick> - <user> - <host> - <channel>
```

Clients might wish to perform a `WHO` query on themselves when connecting to a server, and keep track of numeric `396 RPL_VISIBLEHOST` to keep their `user@host` mask in sync.

Clients might alternatively prefer to use a hard coded limit. A possible worst case scenario for the variable parts might be:

* nick: 20
* user: 20
* host: 63
* channel: 32

Which would leave 353 bytes for each message.

## Server implementation considerations

This section is non-normative.

In the context of flood protection, keep in mind that clients might send an entire batched multiline message all at once.

## Examples

This section is non-normative.

NOTE: In these examples, `<SPACE>` indicates a space character which would otherwise not be clearly visible.

Client sending a mutliline batch

    Client: BATCH +123 draft/multiline #channel
    Client: @batch=123 PRIVMSG #channel hello
    Client: @batch=123 privmsg #channel :how is<SPACE>
    Client: @batch=123;draft/multiline-concat PRIVMSG #channel :everyone?
    Client: BATCH -123

Server sending the same batch

    Server: @msgid=xxx;account=account :n!u@h BATCH +123 draft/multiline #channel
    Server: @batch=123 :n!u@h PRIVMSG #channel hello
    Server: @batch=123 :n!u@h PRIVMSG #channel :how is<SPACE>
    Server: @batch=123;draft/multiline-concat :n!u@h PRIVMSG #channel :everyone?
    Server: BATCH -123

Server sending messages to clients without multiline support

    Server: @msgid=xxx;account=account :n!u@h PRIVMSG #channel hello
    Server: @account=account :n!u@h PRIVMSG #channel :how is<SPACE>
    Server: @account=account :n!u@h PRIVMSG #channel :everyone?

Final concatenated message

    hello
    how is everyone?

---

Invalid multiline batch target

    Client: BATCH +456 draft/multiline #foo
    Client: @batch=456 PRIVMSG #bar hello
    Client: BATCH -456

    Server: :irc.example.com FAIl BATCH MULTILINE_INVALID_TARGET #foo #bar :Invalid multiline target

---

`max-bytes` exceeded multiline batch
    
    Server: CAP * LS :draft/multiline=max-bytes=40000

    Client: BATCH +abc draft/multiline #channel
    Client: @batch=abc PRIVMSG #channel hello
    ... lines that exceeed the max-bytes=40000 limit ...

    Server: :irc.example.com FAIl BATCH MULTILINE_MAX_BYTES 40000 :Multiline batch max-bytes exceeded

`max-bytes` exceeded multiline batch
    
    Server: CAP * LS :draft/multiline=max-lines=10

    Client: BATCH +abc draft/multiline #channel
    Client: @batch=abc PRIVMSG #channel hello
    ... 10 more lines ...

    Server: :irc.example.com FAIl BATCH MULTILINE_MAX_LINES 10 :Multiline batch max-lines exceeded

---

Invalid multiline batch

    Client: BATCH +999 draft/multiline #channel
    ... invalid batch contents ...

    Server: :irc.example.com FAIl BATCH MULTILINE_INVALID :Invalid multiline batch

[message tags]: ../extensions/message-tags.html
[batch]: ../extensions/batch-3.2.html
[standard replies]: ../extensions/standard-replies.html
[message ids]: ../extensions/message-ids.html
[labeled response]: ../extensions/labeled-response.html
[echo message]: ../extensions/echo-message-3.2.html
