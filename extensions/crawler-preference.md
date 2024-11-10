---
title: Crawler Preference
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Sadie Powell"
    email: "sadie@witchery.services"
    period: "2024"
---


## Notes for implementing work-in-progress version

This is a work-in-progress specification.

## Introduction

An IRC crawler is a bot that connects to IRC networks to create a publicly available list of servers and channels. This specification allows IRC networks to express a preference to about whether they can be indexed by a well-behaving IRC crawler.

## The `CRAWLER` ISUPPORT token

Servers can express a crawler preference by adertising the the `CRAWLER`[ISUPPORT][] token. Servers MUST advertise this token with a value. If a crawler encounters the token without a value then it MUST ignore the token.

The value of the `CRAWLER` token is a comma (`,`) (0x2C) separated list of data. Each datum consists of a key which might have a value attached. If there is a value attached, the value is separated from the key by a colon (`:`) (0x3A). That is, `<key>[:<value>][,<key2>[:<value2>][,<keyN>[:<valueN>]]]`. Keys specified in this document MUST only occur at most once.

[ISUPPORT]: https://modern.ircdocs.horse/#feature-advertisement

### The `index` key

The `index` key specifies whether a crawler is allowed to index the network. This key MUST be specified.

The value of this key MUST be either `allow` to allow crawlers to index the network or `deny` to prohibit crawlers from indexing the network.

### The `cooldown` key

The `cooldown` key specifies the time period before a crawler is allowed to reindex a network.

The value of this key MUST be a duration in seconds. If not set this key defaults to 3600 seconds (one hour).

## Examples

Example of a network which wishes to be crawled once per hour:

    S: :irc.example.com 005 alice CRAWLER=index:allow,cooldown:3600

Example of a network which does not wish to be crawled (recheck yearly):

    S: :irc.example.com 005 bob CRAWLER=index:deny,cooldown:31536000
