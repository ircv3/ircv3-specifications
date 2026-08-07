---
title: Autocomplete client tags
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Val Lorentz"
    period: 2020
    email: "progval+ircv3@progval.net"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `+autocomplete-request` and `+autocomplete-response` tag names.
Instead, implementations SHOULD use the `+draft/autocomplete-request`
and `+draft/autocomplete-response` tag names to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification defines two client-only message tags to allow clients to get hints
to complete a message their user is writing from a bot.

## Motivation

Many text interfaces provide some form of autocompletion, like nicks in IRC clients,
or file names in shells, to allow users to write their messages/commands faster
and with less mistakes.
Some shell scripts also offer more advanced command-specific completion, such
as bash-completion or oh-my-zsh.

This tag makes it possible for IRC clients to provide similar advanced completion
while their user is writing a command for an IRC bot.

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the [`message-tags`](../extensions/message-tags.html) capability with the server.
Additionally, clients sending `+draft/autocomplete-response` (bots) MUST support the [`+draft/reply`](./reply.html) client tag
(therefore, they MUST support [`msgid`](../extensions/message-ids.html) as well).
Clients sending only `+draft/autocomplete-request` (user-agents or bots) do not need to support [`+draft/reply`](./reply.html)

### Format

An autocompletion request is represented by a [`TAGMSG`][tags] command with a `autocomplete-request` tag using the client-only prefix `+`.
Its value has no restriction, and represents a partial command that should be completed by a bot.

Clients receiving a `autocomplete-request` may reply with a `TAGMSG` whose target is the nick of the sender of the `autocomplete-request`.
It MUST have two tags:

* `+draft/reply=<msgid>` where `<msgid>` is the message id of the request they are responding to
* `+draft/autocomplete-response=<responses>`

The autocomplete-response format in pseudo-BNF is:

```
<responses> ::= [<response> [<TAB> <response>]*]?
<response>  ::= [<BS>]* <response_string>
<response_string> ::= <sequence of zero or more utf8 characters except NUL, BS, TAB, CR, LF, semicolon (`;`) and SPACE>
```

where:

* `BS` is the backspace character (`0x08`)
* `TAB` is the tabulation character (`0x09`)
* the content of a `<response_string>` should be encoded like the one of `<escaped_value>`
in the [`message-tags`](../extensions/message-tags.html) specification.

TODO: note that \n is allowed; so this should work for multiline messages,
as long as they fit in the tag? what should we do if they are larger?

TODO: should we limit it to just one response?

TODO: explain that backspaces remove characters. or should bots reply with the entire message instead of editing it?

## Client implementation considerations

This section is non-normative

### Sending an autocomplete request

Clients will probably want to send an autocomplete-request when a user triggers
the client's nick completion mechanism (usually the tab key) and there is no
matching nick, or their own command completion mechanism.

The content of the autocomplete request should be the entire message the user
is writing, up to the place that should be complete (usually the end of the line or
the position of the cursor)

### Sending an autocomplete response

TODO: what is there to say

### Applying an autocomplete response

Upon reception, a user-agent may offer the various responses to the user in a way
that lets them pick one.
They may want to show which bot replied what.

They should read the value of `+draft/reply` to find which request the bot is responding
to (so as not to apply autocompletes to old requests).

When a user picks one of the option, the message being typed should be edited by
removing as many characters as there are backspaces characters and appending
the rest of the response string.

### Privacy consideration

TODO: blah blah, you know the drill.
TODO: should we recomment clients send requests only to nicks (eg. send to XXX if a message starts with "XXX:")?

## Examples

### Basic example

In this example, a user wrote "Bot: disconnect freen" in #channel and presses TAB. Its user agent will send:

    @+draft/autocomplete-request=Bot:\sdisconnect\sfreen TAGMSG #channel

Other clients in the channel will receive:

    @msgid=123;+draft/autocomplete-request=Bot:\sdisconnect\sfreen :nick!user@host TAGMSG #channel

A bot with a "disconnect" command will reply:

    @+draft/reply=123,+draft/autocomplete-response=ode :nick!user@host TAGMSG #channel

and the user agent will transform the current message to: "disconnect freenode"

### Backspaces

In this example, a user started writing a command with a typo "Bot: disconnect frenod" in #channel and presses TAB. Its user agent will send:

    @+draft/autocomplete-request=Bot:\sdisconnect\sfrenod TAGMSG #channel

A bot with a "disconnect" command and understanding the user meant "freenode" but missed an "e" will reply:

    @+draft/reply=123,+draft/autocomplete-response=<BS><BS><BS>enode :nick!user@host TAGMSG #channel
