---
title: "`chghost` Extension"
layout: spec
redirect_from:
  - /specs/extensions/chghost-3.2.html
copyrights:
  -
    name: "Christine Dodrill"
    period: "2013"
    email: "me@christine.website"
  -
    name: "Ryan"
    period: "2016"
    email: "ryan@hashbang.sh"
  -
    name: "James Wheare"
    period: "2020"
    email: "james@irccloud.com"
---

## Introduction

The `chghost` capability allows servers to send a notification when clients change their username or host. This mechanism avoids simulating a reconnect of the client. This is useful for servers that implement virtual hosts or cloaks, and helps to reduce clutter in client UI.

## The `chghost` capability

When a client username or host is changed, servers MUST send the `CHGHOST` message to other clients who share channels with the target client and who have enabled the `chghost` capability. Servers SHOULD also send the `CHGHOST` message to the client whose own username or host changed, if that client also supports the `chghost` capability.

## The `CHGHOST` message

The `CHGHOST` message is as follows:

    :nick!old_user@old_host.local CHGHOST new_user new_host.local

The `new_user` parameter represents the user's username.

The `new_host.local` parameter represents the user's hostname.

One or both of the username and hostname can change during the CHGHOST process.

Servers that implement ident checking might choose to prefix a username with a tilde `~` character to indicate missing confirmation from an ident server. This MUST be considered part of the username and included in any CHGHOST messages where relevant.

    :nick!~old_user@old_host.local CHGHOST ~new_user new_host.local

## Fallback when the `chghost` capability has not been negotiated

When the capability is not enabled for other clients who share channels with the changed client, servers SHOULD send fallback messages to simulate the client reconnecting. This allows clients to keep their user state up to date. For shared channels, the simulated events SHOULD include appropriate `QUIT`, `JOIN` and `MODE` commands, to restore membership and user channel modes.

    :nick!old_user@old_host.local QUIT :Changing hostname
    :nick!new_user@new_host.local JOIN #ircv3
    :ircd.local MODE #ircv3 +v :nick

## Examples

In this example, `tim!~toolshed@backyard` gets their username changed to `~b` and their hostname changed to `ckyard`. Their new user mask is `tim!~b@ckyard`:

    :tim!~toolshed@backyard CHGHOST ~b ckyard

In this example, `tim!b@ckyard` gets their username changed to `toolshed` and their hostname changed to `backyard`. Their new user mask is `tim!toolshed@backyard`:

    :tim!b@ckyard CHGHOST toolshed backyard

## Errata

* Previous versions of this specification did not include any examples, which made it unclear as to whether the de-facto `~` prefix should be included on CHGHOST messages. The new examples make clear that it should be included.
* Previous versions of this specification did not specify whether or not the client whose own user or host changed should receive the message.
