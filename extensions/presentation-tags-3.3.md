---
title: IRCv3.3 Presentation Tags
layout: spec
copyrights:
  -
    name: "Kiyoshi Aman"
    period: 2016
    email: "kiyoshi.aman@gmail.com"
---
## Introduction

This specification concerns adding an optional out-of-band message tag to messages
using the message tagging framework.  Message intent tags should be considered an
initial step in replacing CTCP, as well as providing strong replacements for several
interpretations of `NOTICE's` intent.

## Rationale

### CTCP intent encapsulation

CTCP provides a way of encapsulating messages with intent information.  This is done
by encapsulating the message body inside 0x1 bytes, with the first word in the message
body being a keyword which identifies the intent of the message, such as "ACTION" or
"DCC".

One purpose of this specification is to obsolete this encoding method using the message
tagging framework included in IRCv3.

### `NOTICE` meanings

A lot of clients, and some servers, assign various meanings to `NOTICE`, in addition to
the RFC 1459 meaning of "please do not reply to this message". This specification provides
ways to apply intentions to any message, not just `NOTICE`.

## The `intents` capability

A client MUST request the `intents` capability in order to receive intent tags.  IRCd
software MAY represent intent tagged messages in the legacy encapsulation format described
above for clients which did not request the `intents` capability.

## Presentation tags

### The `intent` Tag

The `intent` tag carries the sender's intentions on how the message should be interpreted.
The following intentions SHOULD be supported:

| Intention   | Meaning                                        |
|-------------|------------------------------------------------|
| `action`    | This message is an 'action'.                   |
| `announce`  | Mark this message as an announcement.          |
| `error`     | This message is an 'error' notification.       |
| `noreply`   | Do not generate a reply.                       |

The `action` intent MAY be translated to and from the legacy CTCP encoding.

### The `redirect` tag

The `redirect` tag specifies a target context which is different from the usual context. The
format for the `redirect` tag is as follows:

    redirect=<Target>

The `<Target>` must be a valid channel name.

### The `log` Tag

The `log` tag MAY carry a string indicating a severity. It is intended primarily for automatic
reporting of server state. The format is as follows:

    log=<String>

### The `name` Tag

The `name` intent allows a user to represent a given message as being associated with a
different name. In order to prevent abuse, implementations SHOULD provide mechanisms to prevent
users from masquerading as server or channel operators, and clients MUST visually associate the
user's actual handle with the masqueraded handle. It has the following format:

    name=<String>

### Examples

`@intent=action PRIVMSG #example :dances a jig.`

`@intent=announce,noreply PRIVMSG #example :Please be advised that this is a test.`

`@intent=noreply,announce;redirect=#example PRIVMSG user :Follow the rules at...`

`@log=info NOTICE user :Foobar opered up.`

## Legacy translation

The IRC server MAY translate intents to the legacy encoding and MAY translate intents
from the legacy CTCP encoding as well.

During the transition period, the IRC server SHOULD perform this translation until a
point in time is reached where translation is no longer necessary.

## Metadata vs. Intents

Some CTCP intents were used for the purpose of querying metadata.  Instead, the IRCv3
metadata framework should be used instead.  The Intents framework should be used strictly
for messages which need to be reinterpreted or active messages instead of queries.
