---
title: UTF8ONLY ISUPPORT token
layout: spec
meta-description: An ISUPPORT token that indicates only UTF-8 traffic is allowed.
copyrights:
  -
    name: "Daniel Oaks"
    period: "2021"
    email: "daniel@danieloaks.net"
  -
    name: "Shivaram Lingamneni"
    period: "2021"
    email: "slingamn@cs.stanford.edu"
---

## Introduction
IRC predates the Unicode standard. Consequently, although UTF-8 has been widely adopted on IRC, clients cannot assume that all IRC data is UTF-8. This specification defines a way for servers to advertise that they only allow UTF-8 on their network, letting clients change their processing of outgoing and incoming messages accordingly.

## The `UTF8ONLY` ISUPPORT token
This specification introduces a new token `UTF8ONLY` that servers can include in their [ISUPPORT](https://modern.ircdocs.horse/#feature-advertisement) (`005`) output. Servers publishing this token MUST NOT relay content (such as `PRIVMSG` or `NOTICE` message data, channel topics, or realnames) containing non-UTF-8 data to clients. Clients implementing this specification MUST NOT send non-UTF-8 data to the server once they have seen this token. Server handling of such messages is implementation-defined; for example, they MAY send the `FAIL` code described below, disconnect the client with an `ERROR` message, or respond in some other way.

If a client implementing this specification sees this token, they MUST set their outgoing encoding to UTF-8 without requiring any user intervention. This allows clients to work transparently on networks that only allow UTF-8 traffic.

## The `INVALID_UTF8` `FAIL` code
This is a code that can be used with the `FAIL` command, as defined by the [standard replies](https://ircv3.net/specs/extensions/standard-replies) specification. This code indicates to the client that their message was dropped or modified because it contained non-UTF-8 bytes.

## Examples

```
Client: PRIVMSG #ircv3 :<non-utf-8 message>
Server: FAIL PRIVMSG INVALID_UTF8 :Message rejected, your IRC software MUST use UTF-8 encoding on this network
```

```
Client: USER u s e :<non-utf8 realname>
Server: FAIL USER INVALID_UTF8 :Message rejected, your IRC software MUST use UTF-8 encoding on this network
```

```
Client: PRIVMSG #ircv3 :<non-utf-8 message>
Server: ERROR :Your IRC software MUST use UTF-8 encoding on this network
... server disconnects the client ...
```
