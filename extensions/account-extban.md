---
title: Account Extended Ban
layout: spec
work-in-progress: true
copyrights:
  -
    name: "June McEnroe"
    email: "june@causal.agency"
    period: "2021-2022"
---


## Notes for implementing work-in-progress version

This is a work-in-progress specification.


## Introduction

The [account-notify](account-notify.html), [account-tag](account-tag.html) and [extended-join](extended-join.html) specifications allow clients to much more easily know the account names of logged in users.
The [EXTBAN][] ISUPPORT token indicates the types of extended ban masks that the server supports, but not their meanings.
This specification allows clients to construct ban masks for accounts by indicating which extended ban type matches account names.

[EXTBAN]: https://modern.ircdocs.horse/#extban-parameter


## The `ACCOUNTEXTBAN` ISUPPORT token

Servers publishing the `ACCOUNTEXTBAN` [ISUPPORT][] token allow clients to construct ban masks matching account names.
Servers publishing the `ACCOUNTEXTBAN` token MUST also publish the `EXTBAN` token.
The value of the `ACCOUNTEXTBAN` token is a comma-separated list of names for the extended ban type which matches account names.
Servers MUST support matching exact account names.
Servers MAY support matching account masks and other server-specific extensions.

[ISUPPORT]: https://modern.ircdocs.horse/#feature-advertisement


## Examples

Example with the [account-tag](account-tag.html) capability enabled:

    S: :irc.ircv3.net 005 alice EXTBAN=$,ARar ACCOUNTEXTBAN=R :are supported by this server
    ...
    S: @account=bob :bob_!~bob1@example.org PRIVMSG #chat :I am bad
    C: MODE #chat +b $R:bob

Example with long extban names:

    S: :irc.ircv3.net 005 alice EXTBAN=~,a ACCOUNTEXTBAN=a,account :are supported by this server
    ...
    C: MODE #chat +b ~account:bob
