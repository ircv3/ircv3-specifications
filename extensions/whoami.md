---
title: "`whoami` Extension"
layout: spec
meta-description: A capability allowing clients to track their NUH ("nick-user-host") strings
work-in-progress: true
copyrights:
  -
    name: "Shivaram Lingamneni"
    period: "2026"
    email: "slingamn@cs.stanford.edu"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `whoami` CAP name. Instead, implementations SHOULD use the `draft/whoami` CAP name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use an unprefixed CAP name.

## Motivation

When a server relays a client's chat message (`PRIVMSG` or `NOTICE`) to other clients, it needs to affix various pieces of metadata in order to ensure that the message is interpreted correctly. A typical relayed message looks like:

```
:nick!user@host PRIVMSG #channel :hello world\r\n
```

In this example, `nick!user@host` is the client's NUH (tuple of nickname, username/ident, and hostname, sent as the "source" or "prefix" of the relayed message), `#channel` is the target, and `hello world` is the message content. The total budget for the relayed message is 512 bytes. In order for the client to ensure that its message can be relayed without truncation or rejection for exceeding the 512-byte limit, it must be able to compute the total length of the metadata that the server will add to the message.

The client typically has direct visibility into the length of the message target, since it sends it in the outgoing message to the server. Furthermore, the client generally has an accurate view of its own nickname as assigned by the server, first from the `001 RPL_WELCOME` line received after connection registration and then from `NICK` lines pushed to the client on any nickname update. This leaves the user and hostname components. At present, there is no robust mechanism for the client to track the values of these components, which are assigned by the server as part of connection registration and may be updated unilaterally by the server at any time. Various workarounds exist in the wild, such as sending `USERHOST` or `WHO` for for the client's own nickname after connection registration, or using the [`chghost`](/specs/extensions/chghost.html) capability. However, all such methods suffer from one or more of the following problems:

* Server implementations may penalize or delay the client for sending the command (`WHO`)
* The response type is not guaranteed to contain the actual username and hostname seen by the server (`USERHOST`)
* There is a window in between completing connection registration and receiving the desired response in which the NUH is not known, necessitating use of a fallback value (`WHO` and `USERHOST`)
* The client is not guaranteed to receive updates about changes to their hostname (`WHO`, `USERHOST`, and even `CHGHOST`, where updates about the client's own NUH are only mandated at the SHOULD level)

## Description

This specification introduces a new capability, `draft/whoami`, and a new command, `WHOAMI`. Clients requesting the capability indicate that they are capable of handling the `WHOAMI` command at any time, including as part of the registration burst. 

The `WHOAMI` command is sent under the following circumstances:

1. If the client requested the capability during connection registration, as part of the registration burst, between the final `005 RPL_ISUPPORT` and the beginning of the `LUSERS` numerics (typically beginning with `251 RPL_LUSERCLIENT`)
2. Any time after connection registration when the client's NUH changes (for example, a nickname change, or the activation or deactivation of a vhost or cloak), whether in response to an explicit client action or not

It has the following syntax:

```
:<source> WHOAMI [realname]\r\n
```

Servers MUST include the `<source>` field, set to the client's current NUH. Servers MUST send the client's realname as a parameter when `WHOAMI` is sent as part of the registration burst, and MAY send it with any other response. A missing final parameter indicates the server's lack of intent to communicate an update about the realname. An empty final parameter has its usual meaning, i.e. it communicates that the client's realname is the empty string.

## Examples

This section is non-normative.

A client requests the capability and receives `WHOAMI` in the registration burst:

```
C: CAP REQ :draft/whoami
S: :ergo.test CAP * ACK draft/whoami
C: NICK alice
C: USER u s e r
C: CAP END
S: :ergo.test 001 alice :Welcome to the ErgoTest IRC Network alice
S: :ergo.test 002 alice :Your host is ergo.test, running version ergo-2.19.0-unreleased-404837245c521d25
S: :ergo.test 003 alice :This server was created Wed, 22 Apr 2026 06:34:41 UTC
S: :ergo.test 004 alice ergo.test ergo-2.19.0-unreleased-404837245c521d25 BERTZios CEIMRUabefhiklmnoqstuv Iabefhkloqv
S: :ergo.test 005 alice AWAYLEN=390 BOT=B CASEMAPPING=ascii CHANLIMIT=#:100 CHANMODES=Ibe,k,fl,CEMRUimnstu CHANNELLEN=64 CHANTYPES=# CHATHISTORY=1000 ELIST=U EXCEPTS EXTBAN=,m EXTJWT=1 FORWARD=f :are supported by this server
S: :alice!~u@bery6muzcsynw.irc WHOAMI r
S: :ergo.test 251 alice :There are 0 users and 1 invisible on 1 server(s)
```

A client changes their nickname and receives `WHOAMI` in addition to `NICK`:

```
C: NICK alice_
S: :alice!~u@bery6muzcsynw.irc NICK alice_
S: :alice_!~u@bery6muzcsynw.irc WHOAMI
```

A client enables a vhost and receives `WHOAMI` reflecting their new hostname:

```
C: PRIVMSG HostServ :SET alice mush.room
S: :alice_!~u@mush.room WHOAMI
```

A server administrator deactivates the client's vhost, causing the client to receive `WHOAMI` without explicit action:

```
S: :alice_!~u@bery6muzcsynw.irc WHOAMI
```

## Implementation considerations

This section is non-normative.

If the client knows its current NUH from parsing `WHOAMI`, it can expect to successfully send messages of length 498 - (len(NUH) + len(target)) bytes. (The 14 missing bytes, relative to the 512-byte limit, are as follows: 1 byte for the `:` character preceding the relayed NUH, 1 byte for the space after it, 7 bytes for the command `PRIVMSG`, 1 byte for the space between command and target, 1 byte for the space between target and message content, 1 byte for the `:` character typically inserted to send the message content as a trailing parameter, and 2 bytes for the final `\r\n`.)

Clients should treat this mechanism as eventually consistent, since there is always a potential window between the server updating the client's NUH and the client receiving the `WHOAMI` update, during which the NUH could be larger than expected. Clients prioritizing reliable delivery of messages may wish to leave a larger safety margin, or even pick a conservative constant upper limit on the size of messages they emit.

Clients wishing to receive ongoing updates about their realname should additionally request the [setname](/specs/extensions/setname) capability.
