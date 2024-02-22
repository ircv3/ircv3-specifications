---
title: Message redaction
layout: spec
work-in-progress: true
copyrights:
  -
    name: "James Wheare"
    email: "james@irccloud.com"
    period: "2020"
  -
    name: "Val Lorentz"
    email: "progval+ircv3@progval.net"
    period: "2023"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `message-redaction` capability name.
Instead, implementations SHOULD use the `draft/message-redaction` capability
name to be interoperable with other software implementing a compatible
work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

This specification enables messages to be deleted.
Use cases include retracting accidentally sent messages, moderation,
and removing a [`+draft/react` client tag][], amongst others.
These are cosmetic use cases and do not provide any operational security
guarantees.

## Architecture

### Dependencies

Clients wishing to use this capability MUST negotiate the [`message-tags`][]
capability with the server.
Clients SHOULD negotiate the [`echo-message`][] capability in order to receive
message IDs for their own messages, so they can be redacted.


### Capability

This specification adds the `draft/message-redaction` capability.
Clients MUST ignore this capability's value, if any.

Implementations that negotiate this capability indicate that they are
capable of handling the command described below.


### Command

To redact a message, a client MUST negotiate the `draft/message-redaction`
capability and send a `REDACT` command to a target nickname or channel.
The command is defined as follows:

    REDACT <target> <msgid> [<reason>]

Where `<msgid>` is the id of the message to be redacted, which MUST be a
`PRIVMSG`, `NOTICE`, or `TAGMSG`.

An optional `<reason>` MAY be provided. As the last parameter, it MAY contain spaces.
If the client is authorised to delete the message, the server:

* SHOULD forward this `REDACT`, with an appropriate prefix, to the target
  recipients that have negotiated the `draft/message-redaction` capability, in the
  same way as PRIVMSG messages.
* MUST not forward this `REDACT` to target recipients that have not negotiated
  this capability (see "Fallback" below)

### Chat history

After a message is redacted, [`chathistory`][] responses SHOULD either:

* exclude it entirely
* replace its content and/or tags with a placeholder and
  add the `REDACT` message to the response (not counting toward message limits)
  after the redacted message
* add the `REDACT` message to the response (not counting toward message limits)
  after the redacted message

### Errors

This specification defines `FAIL` messages using the [standard replies][]
framework for notifying clients of errors with message editing and deletion.
The following codes are defined, with sample plain text descriptions.

* `FAIL REDACT INVALID_TARGET <target> :You cannot delete messages from <target>`
* `FAIL REDACT REDACT_FORBIDDEN <target> <target-msgid> :You are not authorised to delete this message`
* `FAIL REDACT REDACT_WINDOW_EXPIRED <target> <target-msgid> <window> :You can no longer edit this message`
* `FAIL REDACT UNKNOWN_MSGID <target> <target-msgid> :This message does not exist or is too old`

## Client implementation considerations

It is strongly RECOMMENDED that clients provide visible redaction history to users.
This helps ensure accountability, and mitigates abuse through malicious or
surreptitious redaction. This could be done via a tool tip, or a separate log.
Redacted messages MAY be hidden entirely from the primary message log,
but a deletion log SHOULD be made available.

For the purposes of user interface, clients MAY assume that their own messages
are redactable.
However, this will not always be the case, and there could be other messages
that they have permission to act on.
Pending a mechanism for discovering redaction permissions, clients SHOULD
allow users to attempt to delete any message via some mechanism.

Clients SHOULD NOT provide a default reason if users do not provide one.

When a `REDACT` command's `msgid` parameter references a known message not in
the `target`'s history, clients MUST ignore it.
This allows servers to safely relay `REDACT` commands targeting messages which they
did not keep in their history.

## Server implementation considerations

This section is non-normative.

A key motivation for specifying this capability as a server tag, rather than
a client-only message tag, is to enable more granular redaction permissions.
Clients might be able to determine which messages are their own, but other
use cases would not be feasible without server validation.

Such use cases might include:

* Allowing channel moderators or server admins to delete unwelcome messages from others
* Specifying a cut-off time after which message edits are no longer allowed

If a message is redacted while a client is not present in a channel, servers may send the `REDACT` command in a `chathistory` batch when it re-joins the channel.

If servers use predictable or guessable `msgid`s, they should consider whether errors
returned on `REDACT` may leak a message's existence to users who did not receive it
(in a channel they are/were not in or in private messages).

### Message validation

To implement validation, servers require a mechanism for determining the permissions of
a particular edit or delete action.
The user requesting the action would need to be compared against properties of
the message, given only the message ID and target.

Servers with message history storage could look up the message properties from the ID,
but this might not be possible or desirable in all cases.
Another mechanism could involve encoding any required properties within the message ID
itself, e.g. the account ID, timestamp, etc. Servers might choose to encrypt this
information if it isn't usually public facing. Any information encoded in a message ID
is still opaque and not intended to be parsed by clients.

### Fallback

Server implementations might choose to inform clients that haven't negotiated
the capability that a deletion has taken place.
The fallback method used (if any) is left up to server implementations, but
could take the form of a standard NOTICE or PRIVMSG with information about the
action.
It might be preferable to use relative time descriptions if referring to
messages in the past, for example:

    :irc.example.com NOTICE #channel :nickname redacted a message from othernick from 5 seconds ago: spam

Implementations might also choose not to send a fallback, if this behaviour
is considered too noisy for users.

## Security considerations

The ability to delete messages does not offer any information or operational
security guarantees.
Once a message has been sent, assume that it will remain visible to any
recipients or servers, whether or not it is subsequently redacted.
Above all else, clients that do not support this specification will not see
any changes to the original message.

## Examples

Deleting a PRIVMSG:

    C: PRIVMSG #channel :an example
    S: @msgid=123 :nick!u@h PRIVMSG #channel :an example
    C: REDACT #channel 123 :bad example
    S: :nick!u@h REDACT #channel 123 :bad example

Deleting a TAGMSG:

    C: @draft/react=🤞TAGMSG #channel
    S: @msgid=123;draft/react=🤞TAGMSG #channel
    C: REDACT #channel 123
    S: :nick@u@h REDACT #channel 123

Deleting someone else's PRIVMSG:

    C1: PRIVMSG #channel :join my network for cold hard chats
    S:  @msgid=123 :nick!u@h PRIVMSG #channel :join my network for cold hard chats
    C2: REDACT #channel 123 spam
    S:  :chanop!u@h REDACT #channel 123 spam


[`echo-message`]: ../extensions/echo-message.html
[`+draft/react` client tag]: ../client-tags/react.html
[standard replies]: ../extensions/standard-replies.html
[`message-tags`]: ../extensions/message-tags.html
[`msgid`]: ../extensions/message-ids.html
[`chathistory`]: ../extensions/chathistory.html
