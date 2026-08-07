---
title: RELAYMSG extension
layout: spec
work-in-progress: true
copyrights:
  - name: "James Lu"
    email: "james@overdrivenetworks.com"
    period: "2020"
  - name: "Daniel Oaks"
    email: "daniel@danieloaks.net"
    period: "2020"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `relaymsg` capability name. Instead, implementations SHOULD
use the `draft/relaymsg` capability name to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.

## Introduction

The `relaymsg` extension allows bots to send channel messages using spoofed nicks. The goal is to let relay bots operate transparently (i.e. no sender prefixes for messages sent from a single bot), without the overhead of tracking all remote users. Currently, relayers that wish to create a transparent experience must either connect extra clients to IRC or create users from a services server; however, these methods do not scale well for larger communities.

## Architecture

### The `draft/relaymsg` capability

The `draft/relaymsg` capability lets clients use the `RELAYMSG` command and receive `draft/relaymsg` tags.

If this capability has a value, the given characters are 'nickname separators'. These characters aren't allowed in normal nicknames, and if given one MUST be present in spoofed nicknames. For example, with `draft/relaymsg=/` the spoofed nickname MUST include the character `"/"`.

### `RELAYMSG` command

The `RELAYMSG` command takes the following arguments:

```
RELAYMSG <channel> <spoofed nick> :<message>
```

Upon receiving this command, the IRCd will translate the message to a `PRIVMSG`:

```
@draft/relaymsg=<botnick> :spoofednick!<ident>@<host> PRIVMSG <channel> :<message>
```

Clients MUST request the `draft/relaymsg` capability before using this command. The capability also lets clients receive the `draft/relaymsg` message tag, which is set to the nick of the sender. This allows the sender to distinguish relayed messages from those sent by other clients, preventing forwarding loops in the case of relay bots.

The `ident` and `host` fields are defined by the server implementation and may be configurable.

## Abuse prevention

In order to prevent abuse, servers should take steps to verify that the caller is authorized to use `RELAYMSG`, and that the spoofed nick cannot be used to confuse users. This may include:

- Restricting `RELAYMSG` to IRC operators, channel operators, or other approved users
- Defining a nickname separator (described above) and requiring that spoofed nicks contain one
- Checking that the spoofed nick is not in use
- Checking that the sender is in the target channel

Servers MUST also sanitize or reject nicks that contain reserved IRC characters, including `!+%@&#$:'"?*,.` and whitespace. This is to avoid sending invalid messages to clients.

Servers may choose to filter spoofed nicks further, or pass them through as is. Spoofed nicks do not necessarily need to be valid IRC nicks; implementations may choose to accept UTF-8 text, for example. One benefit to having spoofed nicks always be invalid is preventing IRC users from changing their nick to one of those that the bot is using.

## Example implementations

### Server side

- InspIRCd 3.x via custom module: https://github.com/overdrivenetworks/inspircd-contrib/blob/relaymsg/3.0/m_relaymsg.cpp
- Oragono 2.4.0+

### Client side

- matterbridge fork: https://github.com/overdrivenetworks/matterbridge/blob/for-upstream/relaymsg/bridge/irc/irc.go
