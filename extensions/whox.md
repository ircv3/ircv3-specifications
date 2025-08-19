---
title: WHOX
layout: spec
copyrights:
  -
    name: "Simon Ser"
    period: "2021"
    email: "contact@emersion.fr"
---

## Introduction

The WHOX extension allows clients to request additional fields in the `WHO`
response (e.g. account name) or omit default fields (e.g. username and
hostname).

## `ISUPPORT` token

Servers supporting this specification MUST include the `WHOX` token in
`RPL_ISUPPORT` without any value. Clients MUST accept `WHOX` tokens with a
value for forwards compatibility.

## Extended `WHO` command

The `WHO` command is extended with an optional argument:

    WHO <mask> [%<fields>[,<token>]]

The second argument contains a percent character followed by a list of fields
requested by the client. Each field is represented by a letter. The standard
fields are:

- `t`: return the `<token>` specified by the client
- `c`: return an arbitrary channel the client is joined to
- `u`: return the username
- `i`: return the IP address
- `h`: return the hostname
- `s`: return the server name
- `n`: return the nickname
- `f`: return the `WHO` flags (away, server operator, etc)
- `d`: return the hop count (distance)
- `l`: return the number of seconds the user has been idle for
- `a`: return the account name
- `o`: return the channel op level
- `r`: return the realname

Servers MAY support additional non-standard fields. Servers MUST NOT rely on
the ordering of the fields.

Clients can also specify a token which will be returned by the server in the
replies. The token MUST contain only digit characters and MUST contain at most
3 characters. Clients MUST NOT include 't' in `<fields>` without specifying a
token.

Servers MAY support additional non-standard flags before the percent character.

The server will reply with zero, one or more `RPL_WHOSPCRPL` replies, followed
by a final `RPL_ENDOFWHO` reply. Servers MUST NOT send `RPL_WHOREPLY` replies.

## `RPL_WHOSPCRPL` (354) numeric reply

    :<server> 354 <client> [token] [channel] [user] [ip] [host] [server] [nick] [flags] [hopcount] [idle] [account] [oplevel] [:realname]

Servers MUST send the fields in the order specified above. Servers MUST include
in their reply all of the standard fields requested by the client, and MUST NOT
add any additional field not requested by the client. Servers MUST ignore any
non-standard field they don't support.

Exactly the fields requested by the client MUST be returned by the server. When
the server omits a field the client has requested, the following placeholders
MUST be used:

- `*` is used for `[channel]` when no channel is returned.
- `255.255.255.255` is used for `[ip]` when the server refuses to disclose the
  IP address.
- `0` is used for `[account]` when the user is logged out.
- Any syntactically correct placeholder can be used for `[oplevel]` when the
  server doesn't support op levels, for instance `n/a`.

Servers MAY return multiple or all applicable prefixes in the `[flags]` (`f`)
field, regardless of whether [`multi-prefix`](multi-prefix.html) is enabled.

Clients SHOULD ignore the values of the hop count (`d`) and the channel op
level (`o`) fields, because they are ill-defined and unreliable.

## Examples

### Without a token

    WHO cooluser %cuhnar
    :irc.example.org 354 mynick #ircv3 ~cooluser coolhost cooluser coolaccount :Cool User
    :irc.example.org 315 mynick cooluser :End of WHO list

### With a token

    WHO #ircv3 %afnt,42
    :irc.example.org 354 mynick #ircv3 42 cooluser H coolaccount
    :irc.example.org 354 mynick #ircv3 42 cooloper H* cooloper
    :irc.example.org 315 mynick #ircv3 :End of WHO list
