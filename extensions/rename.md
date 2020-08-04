---
title: IRCv3 `rename` Extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Sadie Powell"
    email: "sadie@witchery.services"
    period: "2017-2018"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `rename` capability name. Instead, implementations SHOULD use the `draft/rename` capability name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

Currently there is no reasonable way to rename channels. All existing methods involve closing down the channel and redirecting it to the a new one. This is undesirable as access and ban lists needs to be manually copied. It also causes problems for client logging as history is split across multiple files.

This specification introduces a method of natively renaming channels that does not suffer from these faults.

## Architecture

### Capabilities

This specification introduces the `draft/rename` capability. This capability informs the server that the client is capable of handling `RENAME` messages.

### Messages

This specification adds the `RENAME` command. This command allows a client to instruct the server to change the name of a channel, and in turn allows a server to inform a client that a channel name has been changed.

A channel rename preserves all channel state, such as membership, mode and topic.

The `RENAME` command has between two and three parameters. The first parameter is the current channel name, the second parameter is the new channel name, and the optional third parameter is a reason for renaming the channel.

Server implementations SHOULD NOT allow channels to be converted between types with the `RENAME` message. Client implementations SHOULD be able to handle renaming between channel types.

Implementations MUST allow channels to be renamed while only changing the casing of a channel name.

### Errors

This specification defines `FAIL` messages using the [standard replies][] framework for notifying clients of errors with channel renaming. The following codes are defined:

* `FAIL RENAME CHANNEL_NAME_IN_USE <old-channel> <new-channel>: The channel name is already taken`
* `FAIL RENAME CANNOT_RENAME <old-channel> <new-channel>: The channel cannot be renamed`
* `FAIL JOIN CHANNEL_RENAMED <old-channel> <new-channel>: The channel has been renamed`

If existing error numerics (such as `ERR_CHANOPRIVSNEEDED`, `ERR_NOTONCHANNEL`, `ERR_NEEDMOREPARAMS`) are more appropriate, they SHOULD be used instead.

## Channel redirection

To help clients that weren't present in the channel during the name change, server implementations MAY keep track of renames and send a `FAIL JOIN CHANNEL_RENAMED` message to clients attempting to join the old channel name, for as long as is deemed necessary by the implementation.

## Fallback

Server implementations MUST implement a fallback mechanism to inform clients that have not negotiated the `draft/rename` capability of a channel name change. The mechanism is as follows:

* Send the client a `PART` message for the old channel name, with part message matching the rename reason if given
* Send the client a `JOIN` message followed by the usual messages that would be sent if the client had joined the new channel normally (`RPL_TOPIC`, `RPL_TOPICWHOTIME`, `RPL_NAMREPLY`, `RPL_ENDOFNAMES` etc).

This fallback SHOULD NOT be used if the rename only changes the case of the channel name, as defined by the [casemapping] in use on the server.

If a server is using channel redirection, the `470` numeric (`ERR_LINKCHANNEL`) MAY be used with descriptive free-form text to redirect clients from the old channel to the new channel. This is a more ambiguous response and SHOULD NOT be used when the capability has been negotiated.

If a client sends a valid `RENAME` command without having negotiated the capability, the server SHOULD rename the channel, but use the fallback mechanism to report the name change to that client. The server MAY send `FAIL` messages to such clients when the `RENAME` command fails.

## Security Considerations

This section is non-normative

In server implementations that link with other servers, take measures to prevent channel name collisions. An example of such a method would be to use channel identifiers similar to how user identifiers are used to prevent nickname collisions in server-to-server protocols.

Server implementations might choose to implement a per-channel cooldown system using appropriate error responses, to prevent abuse. For example flooding large channels with the fallback burst.

Server implementations might choose to limit the renaming of channels to privileged individuals in order to prevent abuse, using appropriate error responses.

Server implementations that allow channel registration might choose to prevent registering an old channel name while a channel redirection is in place.

### Examples

This section is non-normative

Renaming a channel with a reason:

    C: RENAME #boaring #boring :Typo fix
    S: :nick!user@host RENAME #boaring #boring :Typo fix

Renaming a channel without a reason:

    C: RENAME #coding #programming
    S: :nick!user@host RENAME #coding #programming

Renaming a channel when the `draft/rename` capability has not been negotiation

    C: RENAME #foo #bar :Changing the channel name
    S: :nick!user@host PART #foo :Changing the channel name
    S: :nick!user@host JOIN #bar
    S: :irc.example.com 332 nick #bar :Topic of the renamed channel
    S: :irc.example.com 333 nick #bar topic-setter 1487418032
    S: :irc.example.com 353 nick #bar :@nick other-nick and-another
    S: :irc.example.com 366 nick #bar :End of /NAMES list

Failing to rename a channel because the user does not have the appropriate privileges:

    C: RENAME #aniki #aneki
    S: :irc.example.com 482 nick #aniki :You must be a channel operator

Failing to rename a channel because the new channel name already exists:

    C: RENAME #evil #good :Don't be evil
    S: :irc.example.com FAIL RENAME CHANNEL_NAME_IN_USE #evil #good :Channel already exists

Failing to rename a channel because you can not convert between channel types:

    C: RENAME #global %local
    S: :irc.example.com FAIL RENAME CANNOT_RENAME #global %local :You cannot change a channel type

Failing to rename a channel because it has been renamed recently:

    C: RENAME #magical-girls #witches
    S: :irc.example.com FAIL RENAME CANNOT_RENAME #magical-girls #witches :This channel has been renamed recently

[standard replies]: ../extensions/standard-replies.html
[casemapping]: https://tools.ietf.org/html/draft-hardy-irc-isupport-00#section-4.1