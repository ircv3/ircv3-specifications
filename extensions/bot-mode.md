---
title: Bot Mode
layout: spec
meta-description: A user mode that indicates a user is a bot.
copyrights:
  -
    name: "Daniel Oaks"
    period: "2021"
    email: "daniel@danieloaks.net"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `bot` tag name. Instead, implementations SHOULD use the
`draft/bot` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction
This is a standardised mode that lets clients mark themselves as bots, and lets other clients see them as bots.

## The `BOT` ISUPPORT token
Servers publishing the `BOT` [ISUPPORT](https://modern.ircdocs.horse/#feature-advertisement) token let clients mark themselves as bots by setting a user mode. The value of the `BOT` token is the mode character which is used to enable bot mode and is also the flag used in `WHO` responses of bots (e.g. `BOT=B` or `BOT=b`).

When a client is marked as a bot, they are given a special numeric as part of their `WHOIS` response, it is indicated in their `WHO` flags, and servers may include the `bot` tag on that client's outgoing messages.

## The `RPL_WHOISBOT` `(335)` numeric

    :<server> 335 <nick> <target> :<message>

This numeric is returned as part of a bot's `WHOIS` reply.

Like other WHOIS reply numerics, `<nick>` is the nick of the sender,
`<target>` the nick of the client being whoised (the bot), and
`<message>` is arbitrary human-readable text.

## The `WHO` bot flag
When a `RPL_WHOREPLY` `(352)` numeric is returned for a bot, the character used as the value of the ISUPPORT `BOT` token is returned in the flags (alongside `H|G`).

## The `draft/bot` tag
The `draft/bot` tag indicates that the given user is a bot. This tag SHOULD be added by the server to all commands sent by a bot (e.g. `PRIVMSG`, `JOIN`, `MODE`, `NOTICE`, and all others). The tag SHOULD also be added by the ircd to all numerics directly caused by the bot. This tag MUST only be sent to users who have requested the `message-tags` capability. Servers MUST NOT send this tag with a value, and clients MUST ignore any value if it exists.

## Examples

The conventional `BOT` ISUPPORT value is `"B"`, but this example uses `"b"` to demonstrate where the value's used:
```
Bot:    NICK robodan
Bot:    USER u * * :Hi, I'm a bot!
Server: :irc.ircv3.net 001 robodan :Welcome to the IRCv3 IRC Network robodan
Server: [ ... ]
Server: :irc.ircv3.net 005 robodan BOT=b CASEMAPPING=ascii CHANNELLEN=64 CHANTYPES=# ELIST=U EXCEPTS EXTBAN=,m :are supported by this server
Server: [ ... ]
Bot:    MODE robodan +b
Server: :robodan!~u@203.0.113.22 MODE robodan +b
Bot:    WHOIS robodan
Server: :irc.ircv3.net 311 robodan robodan ~u 203.0.113.22 * :Hi, I'm a bot!
Server: :irc.ircv3.net 338 robodan robodan ~u@203.0.113.22 203.0.113.22 :Actual user@host, Actual IP
Server: :irc.ircv3.net 379 robodan robodan :is using modes +bi
Server: :irc.ircv3.net 335 robodan robodan :is a Bot on IRCv3
Server: :irc.ircv3.net 317 robodan robodan 24 1614290357 :seconds idle, signon time
Server: :irc.ircv3.net 318 robodan robodan :End of /WHOIS list
Bot:    WHO robodan
Server: :irc.ircv3.net 352 robodan * ~u 203.0.113.22 irc.ircv3.net robodan Hb :0 Hi, I'm a bot!
Server: :irc.ircv3.net 315 robodan robodan!*@* :End of WHO list


Alice:  whois robodan
Server: :irc.ircv3.net 311 alice robodan ~u 203.0.113.22 * :Hi, I'm a bot!
Server: :irc.ircv3.net 319 alice robodan :#chat
Server: :irc.ircv3.net 335 alice robodan :is a Bot on IRCv3
Server: :irc.ircv3.net 317 alice robodan 258 1614290467 :seconds idle, signon time
Server: :irc.ircv3.net 318 alice robodan :End of /WHOIS list
Alice:  who #chat
Server: :irc.ircv3.net 352 alice #chat ~u 198.51.100.103 irc.ircv3.net alice H :0 I'm a human!
Server: :irc.ircv3.net 352 alice #chat ~u 203.0.113.22 irc.ircv3.net robodan Hb :0 Hi, I'm a bot!
Server: :irc.ircv3.net 315 alice #chat :End of WHO list
[ ... ]
Server: @draft/bot :robodan!~u@203.0.113.22 PRIVMSG #chat :Hello! Try typing .help to see my commands!
```

Example of some future value being sent, which the receiving client will ignore and process as a bare `@draft/bot` tag:
```
Server: @draft/bot=someFutureValueHere=2343 :robodan!~u@203.0.113.22 PRIVMSG #chat :Hello! Try typing .help to see my commands!
```
