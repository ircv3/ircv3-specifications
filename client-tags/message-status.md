---
title: IRCv3 `message-status` client tags
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Evan Magaliff"
    period: "2017"
    email: "evan@muffinmedic.net"
---
## Purpose
This specification defines several `message-status` client tags, which allows clients to send and receive various information about the status of messages before and after they are sent.

## Description
The specification includes three client tags for message tracking: `typing`, `delivered`, and `read`. The `typing` tag indicates that a user is currently typing a message to a specific target. `delivered` and `read` provide for delivery and read receipts, respectively, and enable users to know when and by whom a message has been delivered to or read.

For the purpose of this specification, the terms "delivery receipt" and "read receipt" are synonymous with the use of the `delievered` and `read` client tags, respectively.

## Format
All tags are sent as a [`TAGMSG`][tags] with an appropriate [`server-time`][time]. `delivered` and `read` must also include the [`labeled-response` tag][labels] of the original message. `typing` can be either `true` or `false`; `delivered` and `read` contain no values.

### `typing`
The `typing=true` client tag SHOULD be sent by the client when the first character is typed in the text-input field. Upon receipt of the `PRIVMSG` or end of a multiline [`batch`][batch], clients SHOULD assume that `typing=false` (no explicit tag needs to be sent).

If new text is entered into the text-input field following sending of the `PRIVMSG` or [`batch`][batch], the client MUST send a new `typing=true` tag after sending the preceding `PRIVMSG` or end of [`batch`][batch].

Upon clearing of the text-input field by the user (e.g., message cancelled), the client MUST send a `typing=false` tag to notify other users.

After receiving a `typing=true` tag, the client SHOULD assume the state is unchanged until either a `PRIVMSG`,[`batch`][batch] end, or `typing=false` is received.

### `delivered` and `read`
Both client tags follow the same format. Clients MUST send a `delivered` tag immediately upon receipt of the message, including the [`draft/msgid`][id] as a [`labeled-response`][labels]. Clients MUST send a `read` tag when the message is first brought into view (e.g. the correct target with the corresponding message is in focus). If the message is delivered and read at the same time, clients MAY send both tags in the same `TAGMSG` or send `delivered` followed by `read` as separate `TAGMSG`s. Clients SHOULD NOT elect to send only a `read` tag and MUST NOT send `read` before `delivered`.

A `delivered` and `read` tag MUST be sent for each unique `draft/msgid`. If a [`batch`][batch] is received, the client MUST NOT send a single `delivered` and `read` for the entirety of the batch. Clients MAY send `delivered` and `read` tags before a [`batch`][batch] end is received.

Although several [`concat`][concat] ideas have been proposed, no such spec currently exists. This section is subject to alteration in response to formalization of a `concat` specification, particularly with regards to how batches may be handled.

## Examples
Sender begins typing

    [send] @time=YYYY-MM-DDThh:mm:ss.sssZ;@typing=true :nick!ident@host TAGMSG target

Sender sends a single `PRIVMSG`
    
    [send] @draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    [recv] @label=ID;time=YYYY-MM-DDThh:mm:ss.sssZ;delivered :nick!ident@host TAGMSG target
    [recv] @label=ID;time=YYYY-MM-DDThh:mm:ss.sssZ;read :nick!ident@host TAGMSG target

Sender sends a `batch` (initially out of focus, brought into focus after batch is complete)

    [send] :irc.host BATCH +ID target
    [send] @batch=ID;draft/msgid=ID1;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    [recv] @label=ID1;time=YYYY-MM-DDThh:mm:ss.sssZ;delivered :nick!ident@host TAGMSG target
    [send] @batch=ID;draft/msgid=ID2;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    [recv] @label=ID2;time=YYYY-MM-DDThh:mm:ss.sssZ;delivered :nick!ident@host TAGMSG target
    [send] @batch=ID;draft/msgid=ID3;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    [recv] @label=ID3;time=YYYY-MM-DDThh:mm:ss.sssZ;delivered :nick!ident@host TAGMSG target
    [send] :irc.host BATCH -ID
    [recv] @label=ID1;time=YYYY-MM-DDThh:mm:ss.sssZ;read :nick!ident@host TAGMSG target
    [recv] @label=ID2;time=YYYY-MM-DDThh:mm:ss.sssZ;read :nick!ident@host TAGMSG target
    [recv] @label=ID3;time=YYYY-MM-DDThh:mm:ss.sssZ;read :nick!ident@host TAGMSG target

Sender cancels message and clears the text-input field

    [send] @time=YYYY-MM-DDThh:mm:ss.sssZ;@typing=false :nick!ident@host TAGMSG target

## Use Cases
This specification is intended for use on servers and/or channels where tracking message status would be beneficial. Current implementations on similar platforms has proven useful, especially in collaborative team environments. The ability to send delivery receipts can also prove useful when connections are unstable and some messages may be dropped prior to reaching their intended target.

## Limitations
Clients must support the [`message-tags`][tags] and [`server-time`][time] capabilities. [`labeled-response`][labels] is also required for `delivered` and `read` tags. Specific [Privacy Considerations](#privacy-considerations) are addressed below.

This specification is primarily intended for instances where all users are familiar with each other. In large channels of unknown users, the additional client tags add extra data that must be transmitted over the network with little potentential benefit to the users.

## Privacy Considerations
Read receipts introduce additional privacy concerns for users who may not wish to inform others of when they have read their messages. Servers SHOULD implement mechanisms at all levels to allow full control over the ability to send and receive `read` client tags. Network operators SHOULD be able to enable or disable the tag server-wide. Channels SHOULD have mechanism in place to allow operators to prevent this tag channel wide (e.g. a channel mode). Users SHOULD have the option of disabling read receipts across the entire network. IRC services SHOULD provide a setting to disable read receipts for an account.

[batch]: http://ircv3.net/specs/extensions/batch-3.2.html
[concat]: https://github.com/ircv3/ircv3-specifications/issues/208#issuecomment-285516349
[id]: http://ircv3.net/specs/extensions/message-ids.html
[labels]: http://ircv3.net/specs/extensions/labeled-response.html
[tags]: http://ircv3.net/specs/core/message-tags-3.3.html
[time]: http://ircv3.net/specs/extensions/server-time-3.2.html