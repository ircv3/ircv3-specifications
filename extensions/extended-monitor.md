---
title: "`extended-monitor` Extension"
layout: spec
work-in-progress: true
extends:
  - monitor
  - away-notify
  - account-notify
  - chghost
  - setname
copyrights:
  -
    name: "Simon Ser"
    email: "contact@emersion.fr"
    period: "2021"
---


## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `extended-monitor` capability name. Instead, implementations SHOULD
use the `draft/extended-monitor` capability name to be interoperable with other
software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed capability name.


## Description

With the help of extensions such as [`away-notify`](away-notify.html),
[`account-notify`](account-notify.html), [`chghost`](chghost.html) and
[`setname`](setname.html), clients are able to keep other users' metadata
up-to-date with their local state when they share a channel. However, clients
are not able to do so when they don't share a channel.

The [`monitor`](monitor.html) extension allows clients to track when another
user goes offline or comes online. This specification extends MONITOR to also
include AWAY, ACCOUNT, CHGHOST and SETNAME notifications.

The `extended-monitor` capability advertises that the server supports sending
such extended notifcations for monitored nicks. When enabled by the client:

- If `away-notify` is also enabled, the client will get AWAY notifications for
  monitored nicks.
- If `account-notify` is also enabled, the client will get ACCOUNT
  notifications for monitored nicks.
- If `chghost` is also enabled, the client will get CHGHOST notifications for
  monitored nicks.
- If `setname` is also enabled, the client will get SETNAME notifications for
  monitored nicks.
