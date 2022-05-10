---
title: "Web Push Extension"
layout: spec
copyrights:
  - name: "Simon Ser"
    period: "2021"
    email: "contact@emersion.fr"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `webpush` CAP name. Instead, implementations SHOULD use the `draft/webpush` CAP name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use unprefixed CAP names.

## Description

Historically, IRC clients have relied on keeping a TCP connection alive to receive notifications about new events. However, this design has limitations:

- It doesn't bode well with some platforms such as Android, iOS or the Web. On these platforms, the connection to the IRC server can be severed (e.g. when the IRC client isn't in the foreground), resulting in IRC events not received.
- Battery-powered devices aim to avoid any unnecessary wake-up of the modem hardware. IRC connections don't make the difference between messages which may be important to the user (e.g. messages targeting the user directly) and the rest of the messages. As a result messages are frequently sent over the IRC connection, resulting in battery drain.

To address these limitations, various push notification mechanisms have been designed. This specification standardizes an extension for Web Push.

```
      ┌────────────┐              ┌────────────┐
      │            │  Subscribe   │            │
      │            ├─────────────►│            │
      │ IRC client │              │ IRC server │
      │            │              │            │
      │            │              │            │
      └────────────┘              └─────┬──────┘
             ▲                          │
             │                          │
        Push │                          │Push
notification │                          │notification
             │       ┌──────────┐       │
             │       │          │       │
             └───────┤ Web Push │◄──────┘
                     │  Server  │
                     │          │
                     └──────────┘
```

Web Push is defined in [RFC 8030], [RFC 8291] and [RFC 8292].

Although Web Push has been designed for the Web, it can be used on other platforms as well. Web Push provides a vendor-neutral standard to send push notifications.

## Implementation

The `draft/webpush` capability allows clients to subscribe to Web Push and receive notifications for messages of interest.

Once a client has subscribed, the server will send push notifications for a server-defined subset of IRC messages. Each push notification MUST contain exactly one IRC message as the payload, without the final CRLF.

The messages follow the same capabilities and the same `RPL_ISUPPORT` as when the client registered for Web Push notifications.

Because of size limits on the payload of push notifications, servers MAY drop some or all message tags from the original message. Servers MUST NOT drop the `msgid` tag if present.

## `VAPID` ISUPPORT token

If the server supports [Voluntary Application Server Identification (VAPID)][RFC 8292] and the client has enabled the `draft/webpush` capability, the server MUST advertise its public key in the `VAPID` ISUPPORT token. This key can be used to verify notifications upon reception by the Web Push server.

The value MUST be the [URL-safe base64-encoded][RFC 4648 section 5] public key usable with the Elliptic Curve Digital Signature Algorithm (ECDSA) over the P-256 curve. The value MUST NOT change over the lifetime of the connection to avoid race conditions.

## `WEBPUSH` Command

A new `WEBPUSH` command is introduced. It has a case-insensitive subcommand:

    WEBPUSH <subcommand> <params...>

### `REGISTER` Subcommand

The `REGISTER` subcommand creates a new Web Push subscription.

    WEBPUSH REGISTER <endpoint> <keys>

The `<endpoint>` is an URL pointing to a push server, which can be used to send push messages for this particular subscription.

`<keys>` is a string encoded in the message-tag format. The values are [URL-safe base64-encoded][RFC 4648 section 5]. For the `aes128gcm` encryption algorithm, it MUST contain at least:

- One public key with the name `p256dh` set to the client's P-256 ECDH public key.
- One shared key with the name `auth` set to a 16-byte client-generated authentication secret.

If the server has advertised the `VAPID` ISUPPORT token, they MUST use this VAPID public key when sending push notifications. Servers MUST replace any previous subscription with the same `<endpoint>`.

If the registration is successful, the server MUST reply with a `WEBPUSH REGISTER` message:

    WEBPUSH REGISTER <endpoint>

On error, the server MUST reply with a `FAIL` message.

Servers MAY expire a subscription at any time.

### `UNREGISTER` Subcommand

The `UNREGISTER` subcommand removes an existing Web Push subscription.

    WEBPUSH UNREGISTER <endpoint>

Servers MUST silently ignore `UNREGISTER` commands for non-existing subscriptions.

If the unregistration is successful, the server MUST echo back the `WEBPUSH UNREGISTER` message. On error, the server MUST reply with a `FAIL` message.

### Errors

Errors are returned using the standard replies syntax.

If the server receives a syntactically invalid `WEBPUSH` command, e.g., an unknown subcommand, missing parameters, excess parameters, or parameters that cannot be parsed, the `INVALID_PARAMS` error code SHOULD be returned:

```
FAIL WEBPUSH INVALID_PARAMS <command> <endpoint> <message>
```

If the server cannot fullfill a client command due to an internal error, the `INTERNAL_ERROR` error code SHOULD be returned:

```
FAIL WEBPUSH INTERNAL_ERROR <command> <endpoint> <message>
```

[RFC 8030]: https://datatracker.ietf.org/doc/html/rfc8030
[RFC 8291]: https://datatracker.ietf.org/doc/html/rfc8291
[RFC 8292]: https://datatracker.ietf.org/doc/html/rfc8292
[RFC 4648 section 5]: https://www.rfc-editor.org/rfc/rfc4648.html#section-5
