---
title: IRCv3 typing client tag
layout: spec
copyrights:
  -
    name: "MuffinMedic"
    period: "2017"
  -
    name: "James Wheare"
    period: "2020"
    email: "james@irccloud.com"
---
## Introduction

This specification defines a client-only message tag to indicate the current typing status of users in channels and private messages.

## Motivation

This tag provides a means of communicating further context to a conversation. Knowing that someone is typing can allow users to hold off on sending messages that simply ask for feedback or check that someone has seen their message. Conversations can have an increased sense of immediacy when participants are aware that others are in the process of replying.

## Architecture

For the purpose of this specification, a message is defined as a `PRIVMSG`, `NOTICE`, `TAGMSG`, or [`batch`][batch] end.

### Dependencies

Clients wishing to use this tag MUST negotiate the [`message-tags`](../extensions/message-tags.html) capability with the server.

### Format

A typing notification is represented by a [`TAGMSG`][tags] command sent with a `typing` tag using the client-only prefix `+` and possible values of `active`, `paused`, and `done`.

* The `typing=active` notification SHOULD be sent by clients **continuously** while the user is making updates to the text-input field and the text is not a "/slash command".

* The `typing=paused` notification MAY be sent by clients **once** when the user has paused typing in the text-input field and has not cleared their text.

* The `typing=done` notification MAY be sent by clients **once** when the user clears the text-input field without sending a message.

Input event handlers MUST be throttled so that any `typing` notification is not sent within 3 seconds of another one for a given target.

After receiving a typing notification, clients SHOULD assume the sender is still typing until any of the following conditions is met:

* A message is received from the sender
* The sender leaves the channel or quits the server
* A `typing=done` notification is received from the sender
* At least 30 seconds have elapsed since the last `typing=paused` notification was received
* At least 6 seconds have passed since the last `typing=active` notification was received

## Privacy Considerations

This section is non-normative.

Typing status indicators introduce additional privacy concerns for users who may not wish to inform others of when they are creating or replying to a message. Clients are recommended to provide appropriate privacy controls when enabling this feature. Standard implementation guidelines on [message tag moderation considerations][tags] also apply to servers.

## Client implementation considerations

This section is non-normative.

To prevent unnecessary distraction, clients might choose to not display typing notifications for their own nickname, while still sending them to others.

## Examples

This section is non-normative.

Client begins typing

    Client: @+typing=active :nick!ident@host TAGMSG target

Client begins typing and then stops entering new text without clearing the text-input field

    Client: @+typing=paused :nick!ident@host TAGMSG target

Client begins typing then clears the text-input field without sending the message

    Client: @+typing=done :nick!ident@host TAGMSG target

[batch]: ../extensions/batch-3.2.html
[tags]: ../extensions/message-tags.html