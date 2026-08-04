---
title: Message editing and deletion
layout: spec
work-in-progress: true
copyrights:
  -
    name: "James Wheare"
    email: "james@irccloud.com"
    period: "2020"
contributors:
  -
    name: "jesopo"
    email: "jess@jesopo.uk"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `message-edit` or `message-delete` capability names. Instead, implementations SHOULD use the `draft/message-edit` and `draft/message-delete` capability names to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.


## Introduction

This specification enables messages to be edited and deleted. Use cases include typo correction, retracting accidentally sent messages, and moderation, amongst others. These are cosmetic use cases and do not provide any operational security guarantees.

## Architecture

### Dependencies

Clients wishing to use these capabilities MUST negotiate the [`message-tags`](../extensions/message-tags.html) capability with the server. Additionally, this capability relies on messages being sent with the [`msgid`](../extensions/message-ids.html) tag. Clients SHOULD negotiate the [`echo-message`](../extensions/echo-message-3.2.html) and [`labeled-response`](../extensions/labeled-response.html) capabilities in order to receive message IDs for their own messages, to allow them to be edited and deleted.

### Capabilities

This specification adds the `draft/message-edit` and `draft/message-delete` capabilities.

Implementations that negotiate these capabilities indicate that they are capable of handling the respective commands and message tags described below.

### Editing messages

To edit a message, a client MUST negotiate the `draft/message-edit` capability and send an `EDIT` command to a target nickname or channel. The command is defined as follows:

    @<tags> EDIT <target>

And uses the following tags:

* `draft/target-msgid=<msgid>` to indicate the [`msgid`] of the message to be edited
* `draft/edit-text=<new-text>` to indicate the new text of the message

If the client is authorised to edit the message, the server MUST forward this `EDIT` to the target recipients with an appropriate prefix, in the same way as PRIVMSG messages.

### Deleting messages

To delete a message, a client MUST negotiate the `draft/message-delete` capability and send a `DELETE` command to a target nickname or channel. The command is defined as follows:

    @<tags> DELETE <target>

And uses the following:

* `draft/target-msgid=<msgid>` to indicate the [`msgid`] of the message to be deleted

If the client is authorised to delete the message, the server MUST forward this `DELETE` to the target recipients with an appropriate prefix, in the same way as PRIVMSG messages.

### Errors

This specification defines `FAIL` messages using the [standard replies][] framework for notifying clients of errors with message editing and deletion. The following codes are defined, with sample plain text descriptions.

* `FAIL EDIT EDIT_FORBIDDEN <target> <target-msgid> :You are not authorised to edit this message`
* `FAIL DELETE DELETE_FORBIDDEN <target> <target-msgid> :You are not authorised to delete this message`
* `FAIL EDIT INVALID_EDIT <target> :Invalid edit text`
* `FAIL EDIT INVALID_EDIT <target> <target-msgid> :Invalid edit text`
* `FAIL EDIT EDIT_WINDOW_EXPIRED <target> <target-msgid> <window> :You can no longer edit this message`

## Client implementation considerations

It is strongly RECOMMENDED that clients provide visible edit and deletion history to users. This helps ensure accountability, and mitigates abuse through malicious or surreptitious edits. This could be done via a tool tip, or a separate log. Edited messages SHOULD be clearly marked as such. Deleted messages MAY be hidden entirely from the primary message log, but a deletion log SHOULD be made available.

For the purposes of user interface, clients MAY assume that their own messages are editable and deletable. However, this will not always be the case, and there could be other messages that they have permission to act on. Pending a mechanism for discovering editing permissions, clients SHOULD allow users to attempt to edit or delete any message via some mechanism.

## Server implementation considerations

This section is non-normative

A key motivation for specifying this capability as a server tag, rather than a client-only message tag, is to enable more granular editing and deletion permissions. Clients might be able to determine which messages are their own, but other use cases would not be feasible without server validation.

Such use cases might include:

* Allowing channel moderators or server admins to delete unwelcome messages from others
* Specifying a cut-off time after which message edits are no longer allowed

### Message validation

To implement validation, servers require a mechanism for determining the permissions of a particular edit or delete action. The user requesting the action would need to be compared against properties of the message, given only the message ID and target.

Servers with message history storage could look up the message properties from the ID, but this might not be possible or desirable in all cases. Another mechanism could involve encoding any required properties within the message ID itself, e.g. the account ID, timestamp, etc. Servers might choose to encrypt this information if it isn't usually public facing. Any information encoded in a message ID is still opaque and not intended to be parsed by clients.

### Fallback

Server implementations might choose to inform clients that haven't negotiated the capability that an edit or deletion has taken place. The fallback method used (if any) is left up to server implementations, but could take the form of a standard NOTICE or PRIVMSG with information about the action. It might be preferable to use relative time descriptions if referring to messages in the past, for example:

    :irc.example.com NOTICE #channel :nickname edited a message from 5 seconds ago: an example

Implementations might also choose not to send a fallback, if this behaviour is considered too noisy for users.

## Security considerations

The ability to edit and delete messages does not offer any information or operational security guarantees. Once a message has been sent, assume that it will remain visible to any recipients or servers, whether or not it is subsequently edited or deleted. Above all else, clients that do not support this specification will not see any changes to the original message.

## Examples

Editing a message:

    C: PRIVMSG #channel :anex ample
    S: @msgid=123 :nick!u@h PRIVMSG #channel :anex ample
    C: @draft/target-msgid=123;draft/edit-text=an\sexample EDIT #channel
    S: @msgid=124;draft/target-msgid=123;draft/edit-text=an\sexample :nick!u@h EDIT #channel

Deleting a message:

    C: PRIVMSG #channel :an example
    S: @msgid=123 :nick!u@h PRIVMSG #channel :an example
    C: @draft/target-msgid=123 DELETE #channel
    S: @msgid=124;draft/target-msgid=123 :nick!u@h DELETE #channel
