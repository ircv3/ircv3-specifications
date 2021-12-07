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

The `WHO` command allows clients to obtain information about other clients.
However, `WHO` doesn't allow clients to request extra data (e.g. account names)
nor a subset of the data (e.g. omit user and host).

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

- 't': return the `<token>` specified by the client
- 'c': return an arbitrary channel the client is joined to
- 'u': return the username
- 'i': return the IP address
- 'h': return the hostname
- 's': return the server name
- 'n': return the nickname
- 'f': return the `WHO` flags (away, server operator, etc)
- 'l': return the number of seconds the user has been idle for
- 'a': return the account name
- 'r': return the realname

Servers MAY support additional non-standard fields.

Clients can also specify a token which will be returned by the server in the
replies. The token MUST contain only digit characters and MUST contain at most
3 characters. Clients MUST NOT include 't' in `<fields>` without specifying a
token.

Servers MAY support additional non-standard flags before the percent character.

The server will reply with zero, one or more `RPL_WHOSPCRPL` replies, followed
by a final `RPL_ENDOFWHO` reply. Servers MUST NOT send `RPL_WHOREPLY` replies.

## `RPL_WHOSPCRPL` (354) numeric reply

    :<server> 354 <client> [token] [channel] [user] [ip] [host] [server] [nick] [flags] [idle] [account] [:realname]

Exactly the fields requested by the client MUST be returned by the server. When
the server omits a field the client has requested, the following placeholders
are used:

- `*` is used for `[channel]` when no channel is returned
- `255.255.255.255` is used for `[ip]` when the server refuses to disclose the
  IP address
- `0` is used for `[account]` when the user is logged out

## Examples

### Without a token

    WHO cooluser %cuhnar
    :irc.example.org 354 mynick #ircv3 ~cooluser coolhost cooluser coolaccount :Cool User
    :irc.example.org 315 mynick cooluser :End of WHO list

### With a token

    WHO #ircv3 %tnfa,42
    :irc.example.org 354 mynick #ircv3 42 cooluser H coolaccount
    :irc.example.org 354 mynick #ircv3 42 cooloper H* cooloper
    :irc.example.org 315 mynick #ircv3 :End of WHO list
