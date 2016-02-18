---
title: IRCv3.3 Message Tags
layout: spec
copyrights:
  -
    name: "Kiyoshi Aman"
    period: 2016
    email: "kiyoshi.aman@gmail.com"
---
## Introduction

This specification adds a new capability for general message tags support and
a prefix for expressing client-to-client tags.

## Motivation

A new capability is required in order to indicate support for client-to-client
tags. Additionally, some implementations felt constrained to apply filtering
rules to prevent clients from receiving tags they didn't request.

## Architecture

### Capabilities

This specification adds the `message-tags` capability. Clients which request
this capability indicate that they are capable of parsing all tags, regardless
of support for tags specified in other IRCv3.2 or later specifications.

### Tags

Client-to-client tags are tags which the server forwards from the client which
sent them to all other clients which received the relevant event. A
client-to-client tag starts with a plus sign (`+`) and otherwise obeys the
format provided in [IRCv3.2 tags][irctags].

The updated BNF for keys is as follows:

    <key>           ::= [ '+' ] [ <vendor> '/' ] <sequence of letters, digits, hyphens (`-`)>

### Examples

`@+znc.in/network PRIVMSG user :lol`

`@+displayname=user2 PRIVMSG #channel :"..."`

## Security Considerations

There are no security considerations that implementations should note when
implementing this specification.

## Errata

[irctags]: http://ircv3.net/specs/core/message-tags-3.2.html
