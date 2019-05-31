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

Software implementing this work-in-progress specification MUST NOT use the unprefixed `+typing` tag name. Instead, implementations SHOULD use the `+draft/typing` tag name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Description
This specification defines the `typing` client tag, which allows clients to send and receive the current status of when a user is typing a new message prior to sending.

For the purpose of this specification, a message is defined as a user-generated `PRIVMSG`, `NOTICE`, `TAGMSG`, or [`batch`][batch] end.

## Format
A typing notification is represented by a [`TAGMSG`][tags] command sent with a `typing` tag using the client-only prefix `+` and possible values of `active`, `paused`, and `done`.

The typing-active notification SHOULD be sent by clients whenever any update is made to the text-input field and the text is not a "/slash command". If the user begins typing in the text-input field and pauses typing without clearing the text-input field, the client SHOULD send a typing-paused notification until typing is resumed or the text-input field is cleared. If the user clears the text-input field without sending a message, the client SHOULD send a typing-done notification.

Input event handlers SHOULD be throttled so that typing notifications are not sent within 3 seconds of each other for a given target.

After receiving a typing notification, clients SHOULD assume the sender is still typing until either a message is received, the sender leaves the channel or quits the server, or a done-notification is received. If at least 30 seconds have elapsed since the last typing-paused notification was received or at least 6 seconds have passed since the last typing-active notification was received and no typing-paused notification was received, clients SHOULD assume the user is no longer typing.

## Examples
C1 begins typing

    [C1] @+draft/typing=active :nick!ident@host TAGMSG target

C1 begins typing and then stops entering new text without clearing the text-input field

    [C1] @+draft/typing=paused :nick!ident@host TAGMSG target

C1 begins typing then clears the text-input field without sending the message

    [C1] @+draft/typing=done :nick!ident@host TAGMSG target

## Use Cases
This specification is intended for use on servers and/or channels where knowing if another user is typing a message would be useful. Current implementations on similar platforms have proven useful, especially in collaborative team environments.

## Limitations
Clients must support the [`message-tags`][tags] capability. Specific [Privacy Considerations](#privacy-considerations) are addressed below.

This specification is primarily intended for instances where all users are familiar with each other. In large channels of unknown users, the additional client tags add extra data that must be transmitted over the network with little potentential benefit to the users. This specification may also be beneficial in private messages regardless of network size.

## Privacy Considerations
Typing status indicators introduce additional privacy concerns for users who may not wish to inform others of when they are creating or replying to a message. Clients are encouraged to provide appropriate privacy controls when enabling this feature. Standard implementation guidelines on [message tag moderation considerations][tags] also apply to servers.

[batch]: ../extensions/batch-3.2.html
[tags]: ../extensions/message-tags.html