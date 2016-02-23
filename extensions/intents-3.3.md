---
title: IRCv3.3 Message Intents framework
layout: spec
copyrights:
  -
    name: "Kiyoshi Aman"
    period: 2016
    email: "kiyoshi.aman@gmail.com"
---
This specification concerns adding an optional out-of-band message tag to messages
using the message tagging framework.  Message intent tags should be considered an
initial step in replacing CTCP, as well as providing strong replacements for several
interpretations of `NOTICE's` intent.

## Background

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

## The `intent` message tag

The `intent` tag carries the sender's intentions on how the message should be interpreted.
The following intentions SHOULD be supported:

| Intention   | CTCP? | Additional tag      | Meaning                                        |
|-------------|-------|---------------------|------------------------------------------------|
| `action`    | Yes.  | N/A                 | This message is an 'action'.                   |
| `announce`  | No.   | N/A                 | Mark this message as an announcement.          |
| `error`     | No.   | N/A                 | This message is an 'error' notification.       |
| `log`       | No.   | `loglevel=<string>` | This message contains logging information.     |
| `name`      | No.   | `name=<string>`     | See notes below.                               |
| `noreply`   | No.   | N/A                 | Do not generate a reply.                       |
| `redirect`  | No.   | `redirect=<target>` | Output this message to the target context.     |
| `response`  | No.   | N/A                 | This message is a response to one with `<id>`. |

Intentions marked as CTCP MAY be translated to and from the legacy encoding.

The `loglevel` tag SHOULD be included with its intent, but implementations MAY define a default
interpretation when it is omitted. `log` is intended primarily for automatic reporting of
server-state. The `redirect` tag MUST accompany its intent, and the `redirect` intent MUST
honor message-sending permissions defined by the server implementation.

The `name` intent allows a user to represent a given message as being associated with a
different name. In order to prevent abuse, implementations SHOULD provide mechanisms to prevent
users from masquerading as server or channel operators, and clients MUST visually associate the
user's actual handle with the masqueraded handle.

### Examples

`@intent=action PRIVMSG #example :dances a jig.`

`@intent=announce,noreply PRIVMSG #example :Please be advised that this is a test.`

`@intent=redirect,noreply,announce;redirect=#example PRIVMSG user :Follow the rules at...`

`@intent=log;loglevel=info NOTICE user :Foobar opered up.`

## Legacy translation

The IRC server MAY translate intents to the legacy encoding and MAY translate intents
from the legacy CTCP encoding as well.

During the transition period, the IRC server SHOULD perform this translation until a
point in time is reached where translation is no longer necessary.

## Metadata vs. Intents

Some CTCP intents were used for the purpose of querying metadata.  Instead, the IRCv3
metadata framework should be used instead.  The Intents framework should be used strictly
for messages which need to be reinterpreted or active messages instead of queries.
