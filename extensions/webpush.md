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

The `webpush` capability allows clients to subscribe to Web Push and receive
notifications for messages of interest.

Once a client has subscribed, the server will send push notifications for a
server-defined subset of IRC messages. Each push notification MUST contain
exactly one IRC message as the payload, without the final CRLF.

The messages follow the same capabilities and the same `RPL_ISUPPORT` as when
the client registered for Web Push notifications.

Because of size limits on the payload of push notifications, servers MAY drop
some or all message tags from the original message. Servers MUST NOT drop the
`msgid` tag if present.

## `WEBPUSH` Command

A new `WEBPUSH` command is introduced. It has a case-insensitive subcommand:

    WEBPUSH <subcommand> <params...>

### `VAPIDPUBKEY` Subcommand

The `VAPIDPUBKEY` subcommand allows clients to query the [Voluntary Application Server Identification (VAPID)][RFC 8292] public key of the server. This can be used to decrypt notifications upon reception.

The client can query the VAPID public key by sending the command:

    WEBPUSH VAPIDPUBKEY

The server will reply with a [URL-safe base64-encoded][RFC 4648 section 5] public key usable with the Elliptic Curve Digital Signature Algorithm (ECDSA) over the P-256 curve.

    WEBPUSH VAPIDPUBKEY <key>

The VAPID public key sent by the server MUST remain constant over the lifetime of the connection.

### `REGISTER` Subcommand

The `REGISTER` subcommand creates a new Web Push subscription.

    WEBPUSH REGISTER <endpoint> <keys>

The `<endpoint>` is an URL pointing to a push server, which can be used to send push messages for this particular subscription.

`<keys>` is a string encoded in the message-tag format. The values are [URL-safe base64-encoded][RFC 4648 section 5]. It MUST contain at least:

- One public key with the name `p256dh` set to the client's P-256 ECDH Diffie-Hellman public key.
- One shared key with the name `auth` set to a client-generated authentication secret.

The server MUST use the VAPID public key sent as a reply to the `VAPIDPUBKEY` subcommand when sending push notifications.

### `UNREGISTER` Subcommand

The `UNREGISTER` subcommand removes an existing Web Push subscription.

    WEBPUSH UNREGISTER <endpoint>

[RFC 8030]: https://datatracker.ietf.org/doc/html/rfc8030
[RFC 8291]: https://datatracker.ietf.org/doc/html/rfc8291
[RFC 8292]: https://datatracker.ietf.org/doc/html/rfc8292
[RFC 4648 section 5]: https://www.rfc-editor.org/rfc/rfc4648.html#section-5
