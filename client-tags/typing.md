---
title: IRCv3 typing client tag
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
unprefixed `+typing` tag name. Instead, implementations SHOULD use the
`+draft/typing` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Description
This specification defines the `typing` client tag, which allows clients to send and receive the current status of when a user is typing a new message prior to sending.

For the purpose of this specification, a message is defined as a user-generated `PRIVMSG`, `NOTICE`, `TAGMSG`, or [`batch`][batch] end.

## Format
The `typing` tag is sent as a [`TAGMSG`][tags]. The client tag takes no parameters.

The `typing` client tag SHOULD be sent by the client when the first character is typed in the text-input field and after every 3 seconds of a non-empty text-input field to avoid a state change. If new text is entered into the text-input field following sending of the text-input field content, the client MUST send a new `typing` tag after sending the preceding content.

After receiving a `typing` tag, the client SHOULD assume the state is unchanged until either a message end is received, or a period of 3 seconds has elapsed from when the last `typing` client tag was received and none of the previously defined message types were received in the appropriate target.

## Examples
C1 begins typing

    [C1] @+draft/typing :nick!ident@host TAGMSG target

## Use Cases
This specification is intended for use on servers and/or channels where knowing if another user is typing a message would be useful. Current implementations on similar platforms have proven useful, especially in collaborative team environments.

## Limitations
Clients must support the [`message-tags`][tags] capability. Specific [Privacy Considerations](#privacy-considerations) are addressed below.

This specification is primarily intended for instances where all users are familiar with each other. In large channels of unknown users, the additional client tags add extra data that must be transmitted over the network with little potentential benefit to the users. This specification may also be beneficial in private messages regardless of network size.

## Privacy Considerations
Typing status indicators introduce additional privacy concerns for users who may not wish to inform others of when they are creating or replying to a message. Clients are encouraged to provide appropriate privacy controls when enabling this feature. Standard implementation guidelines on [message tag moderation considerations][tags] also apply to servers.

[batch]: http://ircv3.net/specs/extensions/batch-3.2.html
[tags]: http://ircv3.net/specs/core/message-tags-3.3.html