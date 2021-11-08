---
title: Multiline Messages
layout: spec
work-in-progress: true
copyrights:
  -
    name: "James Wheare"
    period: "2019"
    email: "james@irccloud.com"
contributors:
  -
    name: "jesopo"
    email: "jess@jesopo.uk"
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

This specification depends on the [`batch`][] capability which MUST be negotiated to use multiline messages. The order of capability negotiation is not significant and MUST not be enforced.

This specification also uses the [client batch][], [message tags][], and [standard replies][] frameworks.

### Capabilities

This specification adds the `draft/multiline` capability.

Implementations requesting this capability indicate that they are capable of handling the batch type and message tag described below.

The capability MUST be advertised by servers with a REQUIRED value: a comma (`,`) (0x2C) separated list of tokens. Each token consists of a key with an optional value attached. If there is a value attached, the value is separated from the key by an equals sign (`=`) (0x3D). That is, `<key>[=<value>][,<key2>[=<value2>][,<keyN>[=<valueN>]]]`. Keys specified in this document MUST only occur at most once.

Clients MUST ignore every token with a key that they don't understand.

The only defined capability key so far is:

* `max-bytes` - This defines the maximum allowed total byte length of multiline batched content *REQUIRED*
* `max-lines` - This defines the maximum allowed number of lines in a multiline batch content *RECOMMENDED*

### Tags

This specification adds the `draft/multiline-concat` message tag, which has no value and is used to send message lines that extend beyond the usual byte length limit. Its usage is described below.

### Batch types

This specification adds the `draft/multiline` batch type.

In addition to the base batch parameters (reference-tag and type) a multiline batch has one additional parameter, the target recipient.

Multiline batches MUST only contain one or more PRIVMSG lines, or one or more NOTICE lines. These lines MUST all have a target which matches the batch target.

When receiving a well-formed multiline message batch, implementations MUST collect the message content from each line in the batch and wait until the full batch has been received before processing the message. Processing in this context refers to:

* Servers: delivering the batch to the intended recipients
* Clients: displaying the batched message to the user

The combined message value of a multiline batch is defined as the concatenation of the messages from each individual line within the batch. Line messages are joined by a single line feed (`\n`) byte unless the `draft/multiline-concat` message tag is sent, in which case that line's message is directly joined with the previous line's message with no separation.

Each line feed used to join line messages contributes one byte towards the `max-bytes` limit. No line feed is appended to the final line message of a batch.

Servers MUST NOT reject blank lines other than in the following cases:

* Clients MUST NOT send blank lines with the `draft/multiline-concat` tag.
* Clients MUST NOT send messages consisting entirely of blank lines.

### Fallback

When delivering multiline batches to clients that have not negotiated the multiline capability, servers MUST deliver the component messages without using a multiline BATCH.

Servers MUST NOT send blank lines to clients that have not negotiated the multiline capability.

Servers SHOULD maintain the line composition sent by the client instead of combining to a normalised form before re-splitting. This ensures that steps taken to [split long lines](#splitting-long-lines) appropriately are preserved.

Any tags that would have been added to the batch, e.g. message IDs, account tags etc MUST be included on the first message line to be sent. Tags MAY also be included on subsequent lines where it makes sense to do so.

### Errors

Servers MUST use `FAIL` messages from the [standard replies][] framework to notify clients of errors with multiline batches. The command is `BATCH` and the following codes are defined:

* `MULTILINE_MAX_BYTES <limit>`: `max-bytes` limit exceeded
* `MULTILINE_MAX_LINES <limit>`: `max-lines` limit exceeded
* `MULTILINE_INVALID_TARGET <batch-target> <provided-target>`: mismatched PRIVMSG target within a batch
* `MULTILINE_INVALID`: any other error

## Interaction with other specifications

### Message tags ([spec][message tags])

Clients MUST NOT send tags other than `draft/multiline-concat` and `batch` on messages within the batch. In particular, all client-only tags associated with the message must be sent attached to the initial BATCH command.

### Message ids ([spec][message ids])

Servers MUST only include a message ID on the first message of a batch when sending a fallback to non-supporting clients.

### Labeled response ([spec][labeled response]) with echo-message ([spec][echo message])

Servers MUST only include the label tag on the opening BATCH command when replying to a client using `echo-message`.

## Client implementation considerations

This section is non-normative.

### Splitting long lines

When using `draft/multiline-concat` to split longer lines, care is required to ensure messages are displayed appropriately when combined or in the fallback case.

The recommended splitting method is to split long lines between words where possible and leave the space character at the end of the line. This ensures that pairs of words spanning a line concatenation are still space separated, without adding a space to the start of any fallback lines.

Take care not to split in the middle of multi-byte character sequences.

In order to calculate the maximum allowed space before splitting a line, consider the following line:

```
:nick!~user@host PRIVMSG #channel :hello \r\n
```

The parts of a message that must fit within the 512 byte limit are as follows

| part                | syntax                | length   |
|---------------------|-----------------------|----------|
| leading             | <code>:</code>        | 1        |
| nick                | <code>nick</code>     | variable |
| nick/mask separator | <code>!</code>        | 1        |
| user                | <code>~user</code>    | variable |
| user/host separator | <code>@</code>        | 1        |
| host                | <code>host</code>     | variable |
| command             | <code>PRIVMSG</code>  | 7        |
| channel             | <code>#channel</code> | variable |
| message separator   | <code> :</code>       | 2        |
| crlf                | <code>\r\n</code>     | 2        |
| message             | <code>hello</code>    | variable |

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

For server implementations and operators, consider the impact of relaying a large multiline batch. In particular, the potential to exhaust client send queue budgets, which will either result in client disconnection (a denial-of-service attack) or degraded client performance.

Each additional protocol line carries extra metadata beyond the message content, such as the sender's full source mask, the command, and the target. This means that the byte size of a multiline batch sent to clients will typically be a function of max-lines as well as max-bytes. Configuring max-bytes, max-lines, and the send queue length together is recommended to mitigate these issues.

Additionally, in the context of flood protection and throttling of incoming messages, keep in mind that clients might send an entire batched multiline message all at once.

## Examples

This section is non-normative.

NOTE: In these examples, `<SPACE>` indicates a space character which would otherwise not be clearly visible.

Client sending a mutliline batch

    Client: BATCH +123 draft/multiline #channel
    Client: @batch=123 PRIVMSG #channel hello
    Client: @batch=123 PRIVMSG #channel :
    Client: @batch=123 privmsg #channel :how is<SPACE>
    Client: @batch=123;draft/multiline-concat PRIVMSG #channel :everyone?
    Client: BATCH -123

Server sending the same batch

    Server: @msgid=xxx;account=account :n!u@h BATCH +123 draft/multiline #channel
    Server: @batch=123 :n!u@h PRIVMSG #channel hello
    Server: @batch=123 :n!u@h PRIVMSG #channel :
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

This example is also valid if every instance of PRIVMSG is replaced with NOTICE.

---

Invalid multiline batch target

    Client: BATCH +456 draft/multiline #foo
    Client: @batch=456 PRIVMSG #bar hello
    Client: BATCH -456

    Server: :irc.example.com FAIL BATCH MULTILINE_INVALID_TARGET #foo #bar :Invalid multiline target

---

`max-bytes` exceeded multiline batch
    
    Server: CAP * LS :draft/multiline=max-bytes=40000

    Client: BATCH +abc draft/multiline #channel
    Client: @batch=abc PRIVMSG #channel hello
    ... lines that exceeed the max-bytes=40000 limit ...

    Server: :irc.example.com FAIL BATCH MULTILINE_MAX_BYTES 40000 :Multiline batch max-bytes exceeded

`max-lines` exceeded multiline batch
    
    Server: CAP * LS :draft/multiline=max-bytes=40000,max-lines=10

    Client: BATCH +abc draft/multiline #channel
    Client: @batch=abc PRIVMSG #channel hello
    ... 10 more lines ...

    Server: :irc.example.com FAIL BATCH MULTILINE_MAX_LINES 10 :Multiline batch max-lines exceeded

---

Invalid use of a concatenated blank line

    Client: BATCH +abc123 draft/multiline #channel
    Client: @batch=abc123 PRIVMSG #channel :hello<space>
    Client: @batch=abc123;draft/multiline-concat PRIVMSG #channel :
    Client: @batch=abc123 PRIVMSG #channel :there
    Client: BATCH -abc123

    Server: :irc.example.com FAIL BATCH MULTILINE_INVALID :Invalid multiline batch with concatenated blank line

---

Invalid entirely blank message

    Client: BATCH +abc123 draft/multiline #channel
    Client: @batch=abc123 PRIVMSG #channel :
    Client: @batch=abc123 PRIVMSG #channel :
    Client: BATCH -abc123

    Server: :irc.example.com FAIL BATCH MULTILINE_INVALID :Invalid multiline batch with blank lines only

---

Mixing PRIVMSG and NOTICE

    Client: BATCH +792da7 draft/multiline #channel
    Client: @batch=792da7 PRIVMSG #channel :this starts with a PRIVMSG
    Client: @batch=792da7 NOTICE #channel :but ends with a NOTICE
    Client: BATCH -792da7

    Server: :irc.example.com FAIL BATCH MULTILINE_INVALID :Invalid multiline batch

---

Other invalid multiline batch

    Client: BATCH +999 draft/multiline #channel
    ... invalid batch contents ...

    Server: :irc.example.com FAIL BATCH MULTILINE_INVALID :Invalid multiline batch

[client batch]: ../extensions/client-batch.html
[message tags]: ../extensions/message-tags.html
[`batch`]: ../extensions/batch.html
[standard replies]: ../extensions/standard-replies.html
[message ids]: ../extensions/message-ids.html
[labeled response]: ../extensions/labeled-response.html
[echo message]: ../extensions/echo-message.html
