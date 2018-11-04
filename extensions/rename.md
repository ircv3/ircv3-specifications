---
title: IRCv3 `rename` Extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Peter Powell"
    email: "petpow@saberuk.com"
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

This specification adds the `RENAME` C2S and S2C message. This message informs a client that the name of a channel has been changed.

The `RENAME` message has between two and three parameters. The first parameter is the current channel name, the second parameter is the new channel name, and the optional third parameter is a reason for renaming the channel.

Server implementations SHOULD NOT allow channels to be converted between types with the `RENAME` message. Client implementations SHOULD be able to handle renaming between channel types.

### Numerics

| No. | Label               | Format                                         |
| --- | --------------------| ---------------------------------------------- |
| 692 | `ERR_CHANNAMEINUSE` | `<nick> <new-channel> :Channel already exists` |
| 693 | `ERR_CANNOTRENAME`  | `<nick> <old-channel> <new-channel> :<reason>` |

### Examples

Renaming a channel with a reason:

    C: RENAME #boaring #boring :Typo fix
    S: :nick!user@host RENAME #boaring #boring :Typo fix

Renaming a channel without a reason:

    C: RENAME #coding #programming
    S: :nick!user@host RENAME #coding #programming

Failing to rename a channel because the user does not have the appropriate privileges (this example is non-normative):

    C: RENAME #aniki #aneki
    S: :irc.example.com 482 nick #aniki :You must be a channel operator

Failing to rename a channel because the new channel name already exists:

    C: RENAME #evil #good :Don't be evil
    S: :irc.example.com 692 nick #good :Channel already exists

Failing to rename a channel because you can not convert between channel types (this example is non-normative):

    C: RENAME #global %local
    S: :irc.example.com 693 nick #global %local :You cannot change a channel type

Failing to rename a channel because it has been renamed recently:

    C: RENAME #magical-girls #witches
    S: :irc.example.com 693 nick #magical-girls #witches :This channel has been renamed recently

## Implementation Considerations

Server implementations MUST implement a fallback method for legacy clients. This method SHOULD involve sending the legacy client a `PART` message for the old channel name, with the rename reason as the part message if given, followed by the usual messages that would be sent if the legacy client had joined the new channel normally (`JOIN`, `RPL_TOPIC`, `RPL_NAMREPLY`, etc) and finally a `MODE` message to restore any prefix modes that the legacy client has applied to it.

Server implementations MAY implement `JOIN` redirection from the old channel to the new channel for as long as is deemed necessary.

Server implementations MAY implement a cooldown system to prevent abuse.

## Security Considerations

Server implementations that link across a network MUST take measures to prevent channel name collisions. An example of such a method would be to use channel identifiers similar to how user identifiers are used to prevent nickname collisions in server-to-server protocols.

Server implementations MAY limit the renaming of channels to privileged individuals in order to prevent abuse.
