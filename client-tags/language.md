---
title: Language client tag
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
unprefixed `+language` tag name. Instead, implementations SHOULD use the
`+draft/language` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification defines a client-only message tag to indicate the language
of a message.

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the
[`message-tags`](../extensions/message-tags.html) capability with the server.

### Format

The language tag is sent by a client with the client-only prefix `+` and
its value must be a [BCP 47 language tag](https://tools.ietf.org/html/bcp47).

    +draft/language=<language-tag>

In case the value is not a valid language tag, clients SHOULD ignore it.

### Guessing languages

Clients sending messages SHOULD NOT try to guess the language of a message
based only on its content.
However, if they may do some guessing if they have an external hint from
the user or configuration and are reasonably confident in their guess.

This allows receiving clients to trust the language more than if they
guessed it themselves.

### Interactions with +draft/multiline

In case of a multiline message, `+draft/language` tags must be on
the individual parts.

Its value may change between parts in the same batch,
which allows representing a message containing multiple languages
(eg. quoting from a different language, or citing a word).

## Client implementation considerations

This section is non-normative

Clients can use this information to provide translations or dictionary
definitions for words when displaying a message.

## Examples

This section is non-normative.

### Simple message

    C: @+draft/language=en PRIVMSG #channel :Hello to you!
    C: @+draft/language=fr-FR PRIVMSG #channel :Je parle aussi français.

### Multiline message

NOTE: In this examples, `<SPACE>` indicates a space character which would
otherwise not be clearly visible.

    C: BATCH +123 draft/multiline #channel
    C: @batch=123 +draft/language=en PRIVMSG #channel :The French word for "cat" is<SPACE>
    C: @batch=123;draft/multiline-concat;+draft/language=fr-FR PRIVMSG #channel :"chat"
    C: BATCH -123
