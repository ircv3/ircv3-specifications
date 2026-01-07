---
title: "`display-name` client tag"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "James Wheare"
    period: "2021"
    email: "james@irccloud.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `+display-name` tag name. Instead, implementations SHOULD use the
`+draft/display-name` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification defines a client-only message tag to indicate a display name to use for the attached message, separate from the nickname or realname of the user.

## Motivation

This tag provides a way to provide further temporary context about the originator of a message. Example use cases include bots that relay messages from external contexts, or in role playing environments.

## Architecture

For the purpose of this specification, a message is defined as a `PRIVMSG`, `NOTICE`, `TAGMSG`, or [`batch`][batch] end.

### Dependencies

Clients wishing to use this tag MUST negotiate the [`message-tags`][tags] capability with the server.

### Format

Display names are indicated by a message with a `+draft/display-name` tag using the client-only prefix with the value being the desired name.

### Fallback string

Sending clients SHOULD begin their messages with a usable fallback to cater to clients that don't support the display name tag. A common existing method of indicating a display name is to include the desired name at the start of a message, surrounded by angle brackets, backticks, or other forms of parenthesis.

For instance, the following message has a display name tag with the value `charles` and the message begins with the fallback string `<charles> `.

```
@+draft/display-name=charles :bot!bot@bot PRIVMSG #channel :<charles> Hello from outside IRC
```

To support this fallback, receiving clients that support the display name tag SHOULD hide such a prefix if it matches the tag value.

Receiving clients SHOULD check for the presence of formatting codes in the fallback string as well as the U+200B ZERO WIDTH SPACE character, and ignore them when comparing against the display name. These are often used to visually separate the relayed name from the message, and to avoid triggering highlights on users present on both sides of a relay.

## Client implementation considerations

This section is non-normative.

This client tag is not meant as a method of impersonating other IRC users. Take care to make it clear to users when a display name is being used. Consider formatting display names differently to nicknames and making the original nickname visible to the user.

This client tag is also not meant to be used as a persistent display name for users. The [metadata framework][metadata] would be more suitable for such a use case.

## Examples

This section is non-normative.

    Client: @+draft/display-name=Non\sIRC\sUser :bot!bot@bot PRIVMSG #channel :<Non IRC User> Hello from outside IRC
    Client: @+draft/display-name=AnotherUser :bot!bot@bot PRIVMSG #channel :<AnotherUser> Hi from me too!
    Client: @+draft/display-name=NoPrefixUser :bot!bot@bot PRIVMSG #channel :I'm not sending a fallback prefix
    Client: :bot!bot@bot PRIVMSG #channel :Bot message
    Client: :alice!alice@alice PRIVMSG #channel :Hi from me!
    Client: @+draft/display-name=Colourful :bot!bot@bot PRIVMSG #channel :<[U+0003]03C[U+200B]olourful[U+0003]> My fallback nick is in green

Example display from a client that supports the display name tag and performs fallback string removal

```
    <Non IRC User (bot)> Hello from outside IRC
     <AnotherUser (bot)> Hi from me too!
    <NoPrefixUser (bot)> I'm not sending a fallback prefix
                   <bot> Bot message
                 <alice> Hi from me!
       <Colourful (bot)> My fallback nick is in green
```

Example display from a client without support for the display name tag

```
                   <bot> <Non IRC User> Hello from outside IRC
                   <bot> <AnotherUser> Hi from me too!
                   <bot> I'm not sending a fallback prefix
                   <bot> Bot message
                 <alice> Hi from me!
                   <bot> <Colourful> My fallback nick is in green
```


[batch]: ../extensions/batch.html
[tags]: ../extensions/message-tags.html
[metadata]: ../core/metadata-3.2.html
