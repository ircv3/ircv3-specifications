---
title: IRCv3 editmsg extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Evan Magaliff"
    period: "2017"
    email: "muffinmedic@kiwiirc.com"
  -
    name: "Darren Whitlen"
    period: "2017"
    email: "prawnsalad@kiwiirc.com"
---
## Notes for implementing work-in-progress version
This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `delete` and `edit` message tags. Instead, implementations SHOULD use the `draft/delete` and `draft/edit` message tags to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed message tags.

## Purpose
This specifications implements standard message tags to allow for editing and deleting of previously sent messages. This enables users to correct mistakes, alter their messages, and remove content they no longer wish to be displayed to other users.

## Description
This document describes the `editmsg` extension and adds the `delete` and `editmsg` message tags. Clients MUST support the [`server-time`][server-time], [`echo-message`][echo-message], [message-tags][message-tags], [`draft/labeled-response`][draft/labeled-response] and [`draft/msgid`][draft/msgid] capabilities.

When a user edits or deletes a message, a new `draft/msgid` MUST be included and the original `draft/msgid` MUST be sent with the `draft/delete` and `draft/editmsg` tags. The server SHOULD verify the authenticity of the user and SHOULD ensure they have the proper permissions to make the requested changes (see [Security Considerations](#security-considerations)). Once the changes have been verified, the server MUST pass on the updated message to all clients.

Upon a successful message edit or deletion, the server MUST acknowledge the change with `echo-message`.

If the user does not have the appropriate permissions to make the requested changes, the server MUST return an `ACCESS_DENIED` error and MUST NOT forward the requested changes to other clients.

If the edited message contains content not allowed by the IRC server (e.g. a blocked word or formatted text), the IRC server MUST return the same appropriate numeric it would for a `PRIVMSG`.

If a message change is requested for a `draft/msgid` that does not exist in the target, the server MUST return an `MSG_NOT_FOUND` error.

If the edited message exceeds the maximum allowed line length, the client SHOULD break the message up into smaller lines and each line MUST have a unique `draft/msgid`. If the client does not split the message, the server MUST truncate the message to it's maximum allowed length.

## Format

The `editmsg` message tag is used to change the contents of a perviously sent message. The entire updated message MUST be sent, MUST have a unique `draft/msgid`, and MUST contain the original message's `draft/msgid` in a `draft/label` tag. Partial changes MUST NOT be sent.

    @draft/msgid=ORIG_ID PRIVMSG target :My favorite password is hunger2
    :nick!user@host PRIVMSG target :What's hunger2?
    @draft/msgid=NEW_ID;draft/editmsg=ORIG_ID PRIVMSG target :My favorite password is hunter2

The `delete` message tag is used to delete a previously sent message. The client SHOULD NOT include any content in the `PRIVMSG`. The `draft/msgid` of the message to be deleted MUST be included in a `draft/label` tag.

    @draft/msgid=ID PRIVMSG target :Hi! I accidentally sent this to the wrong window.
    @draft/msgid=NEW_ID;draft/delete=OLD_ID PRIVMSG target

`echo-message` aknowledgement MUST be sent for a successful edit or deletion.

    --> @draft/msgid=NEW_ID;draft/delete=OLD_ID PRIVMSG target
    @draft/delete=OLD_ID :nick!user@host PRIVMSG target

Error messages must be sent as stated above. The `draft/editmsg` or `draft/delete` message tag MUST be the `draft/msgid` of the edited message and MUST NOT be the `draft/msgid` of the original message being edited.

    @draft/editmsg=ID :nick!ident@host ERR ERROR_CODE

## Use Cases
This specification is intended for any instance a user wishes to change the contents of a previously sent message or delete a previously sent message. Such examples can include, but are not limited to, spelling and grammar issues a user wishes to correct, posting a bad URL to be fixed, or posting of comprimising infomration they wish to delete.

## Limitations
Clients that do not support `editmsg` or have the feature disabled will receive standard `PRIVMSG` with the updated messages for backwards compatibility for all existing clients but may not properly format updates or deletions (see [Security Considerations](#security-considerations)). This specification is best utilized by IRC services that retain complete control over both server and client functionality where use of this specification can be ensured.

## Security Considerations
This specification does not include limitations on how edit and deletion permissions are granted to users and verified by the server, although specific recommendations follow. Although IRC servers may allow anyone to modify any message, IRC servers SHOULD limit this ability to the user originally sending the message. IRC servers MAY allow network or channel operators to delete messages of other users and SHOULD NOT allow network or channel operators to edit the contents of another user's messages. 

Although clients SHOULD restrict editing to messages the user has sent, it is the responsbility of the IRC servers to verify the authenticity of the requested changes. Servers SHOULD utilize a secure method for verification.

This specification provides standard message tags to improve the usability of IRC from a front-end user perspective. Users MUST NOT rely on this specification to provide privacy or security. Once a message has been sent to an IRC server, users SHOULD assume it has been logged and recorded in it's original state and SHOULD NOT assume that message changes have been applied to all logs, recordings, and clients. To provide transparency to users, clients SHOULD mark appropriate messages as having been edited and SHOULD indicate the user making the edit.

## Alternate Specifications
Alternative specifications are in progress and will be listed here for comparison and discussion.

[echo-message]: http://ircv3.net/specs/extensions/echo-message-3.2.html
[message-tags]: http://ircv3.net/specs/core/message-tags-3.3.html
[server-time]: http://ircv3.net/specs/extensions/server-time-3.2.html
[draft/msgid]: https://github.com/ircv3/ircv3-specifications/pull/285
[draft/labeled-response]: https://github.com/ircv3/ircv3-specifications/pull/162
[multiline]: https://github.com/ircv3/ircv3-specifications/issues/208
[cap]: ../core/capability-negotiation-3.2.html