---
title: IRCv3.3 Presentation Tags
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Kiyoshi Aman"
    email: "kiyoshi.aman@gmail.com"
    period: "2016"
---

## Introduction

Presentation tags are a set of tags which offer presentational cues for
clients. These tags (`intent`, `redirect`, `name`, and `log`) allow server
and bot authors to specify that their messages carry particular meaning in
a standardized way.

## Motivation

Over the years since RFC 1459, several meanings have been imputed to `NOTICE`
beyond that specification's requirement that there be no automatic responses
to messages sent via `NOTICE`. Specialized message formats have also become
a de-facto standard for actions and for instructing clients to display
messages in a particular context. 

## Architecture

### Capabilities

This specification adds the `intents` capability, which enables all of the
tags documented in this specification.

### Tags

#### The `intent` Tag

The `intent` tag carries the sender's intentions on how the message should be
interpreted. The following intentions SHOULD be supported:

| Intention   | Meaning                                               |
|-------------|-------------------------------------------------------|
| `action`    | This message is an 'action'.                          |
| `announce`  | Mark this message as an announcement.                 |
| `info`      | This message is for informational purposes, e.g. news |
| `error`     | This message is an 'error' notification.              |
| `noreply`   | Do not generate a reply.                              |

The `action` intent MAY be translated to and from the legacy CTCP encoding.

#### The `redirect` tag

The `redirect` tag specifies a target context which is different from the usual
context. The format for the `redirect` tag is as follows:

    redirect=<Target>

The `<Target>` must be a valid channel name, and the user must be capable of
sending messages to the `<Target>`.

#### The `log` Tag

The `log` tag MAY carry a string indicating a severity. It is intended
primarily for automatic reporting of server state. The format is as follows:

    log=<String>


### Examples

`@intent=action PRIVMSG #example :dances a jig.`

`@intent=announce,noreply PRIVMSG #example :Please be advised that this is a test.`

`@intent=noreply,announce;redirect=#example PRIVMSG user :Follow the rules at...`

`@log=info NOTICE user :Foobar opered up.

### Legacy translation

The `intent=action`, `intent=noreply`, and `redirect` tags SHOULD be
translated as follows for legacy clients:

* `intent=action` → `\x01ACTION ...`
* `intent=noreply` becomes a `NOTICE`.
* `redirect=<target>` → `NOTICE user :[<target>] ...`

The other direction MUST be done for messages from legacy clients.

## Security Considerations

At present, there are no security concerns to discuss regarding this
specification.

## Errata

There are presently no errata available for this specification.
