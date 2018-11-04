---
title: IRCv3 delivered client tag
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Evan Magaliff"
    period: "2017"
    email: "evan@muffinmedic.net"
---
## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `+delivered` tag name. Instead, implementations SHOULD use the
`+draft/delivered` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use the unprefixed tag name.

## Description
The specification includes the `delivered` client tag for message tracking. Theis tag provides for delivery receipts, and enables users to know when and by whom a message has been delivered to.

For the purpose of this specification, the term "delivery receipt" synonymous with the use of the `delivered` client tag.

For the purpose of this specification, a message is defined as a user-generated `PRIVMSG`, `NOTICE`, `TAGMSG`, or [`batch`][batch] end.

## Format
`delivered` is sent as a [`TAGMSG`][tags] and MUST include the [`draft/msgid`][id] of the original message as the client tag parameter.

Clients MUST send a `delivered` tag immediately upon receipt of the message, including the [`draft/msgid`][id] as the client tag parameter. If the message is delivered and read at the same time, clients MAY send both the `delivered` and `read` tags in the same `TAGMSG` or send `delivered` followed by `read` as separate `TAGMSG`s. Clients SHOULD NOT elect to send only a `read` tag and MUST NOT send `read` before `delivered`.

A `delivered` tag MUST be sent for each unique `draft/msgid`. If a [`batch`][batch] is received, the client MUST NOT send a single `delivered` for the entirety of the batch. Clients MAY send a `delivered` before a [`batch`][batch] end is received. Upon receipt of a `delivered` tag, clients SHOULD assume all previous messages in the given target have also been delivered to the user as appropriate.

Although several [`concat`][concat] ideas have been proposed, no such spec currently exists. This section is subject to alteration in response to formalization of a `concat` specification, particularly with regards to how batches may be handled.

## Examples
C1 sends a single, non-batch message to C2
    
    [C1] @draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    [C2] @+draft/delivered=ID :nick!ident@host TAGMSG target
    [C2] @+draft/read=ID :nick!ident@host TAGMSG target

C1 sends a `batch` to C2 (initially out of focus, brought into focus after batch is complete)

    [C1] :irc.host BATCH +ID target
    [C1] @batch=ID;draft/msgid=ID1;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    [C2] @+draft/delivered=ID1 :nick!ident@host TAGMSG target
    [C1] @batch=ID;draft/msgid=ID2;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    [C2] @+draft/delivered=ID2 :nick!ident@host TAGMSG target
    [C1] @batch=ID;draft/msgid=ID3;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    [C2] @+draft/delivered=ID3 :nick!ident@host TAGMSG target
    [C1] :irc.host BATCH -ID
    [C2] @+draft/read=ID1 :nick!ident@host TAGMSG target
    [C2] @+draft/read=ID2 :nick!ident@host TAGMSG target
    [C2] @+draft/read=ID3 :nick!ident@host TAGMSG target

## Use Cases
This specification is intended for use on servers and/or channels where tracking message status would be beneficial. Current implementations on similar platforms has proven useful, especially in collaborative team environments. The ability to send delivery receipts can also prove useful when connections are unstable and some messages may be dropped prior to reaching their intended target.

## Limitations
Clients must support the [`message-tags`][tags] capability. Specific [Privacy Considerations](#privacy-considerations) are addressed below.

This specification is primarily intended for instances where all users are familiar with each other. In large channels of unknown users, the additional client tags add extra data that must be transmitted over the network with little potentential benefit to the users. This specification may also be beneficial in private messages regardless of network size.

## Privacy Considerations
Clients are encouraged to provide appropriate privacy controls when enabling this feature. Standard implementation guidelines on [message tag moderation considerations][tags] also apply to servers.

[batch]: http://ircv3.net/specs/extensions/batch-3.2.html
[concat]: https://github.com/ircv3/ircv3-specifications/issues/208#issuecomment-285516349
[id]: http://ircv3.net/specs/extensions/message-ids.html
[tags]: http://ircv3.net/specs/core/message-tags-3.3.html