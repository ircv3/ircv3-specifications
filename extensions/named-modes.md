---
title: IRCv3 Named Modes Extension
layout: spec
copyrights:
  -
    name: "Attila Molnar"
    period: "2014"
    email: "attilamolnar@hush.com"
  -
    name: "Sadie Powell"
    period: "2017"
    email: "sadie@witchery.services"
  -
    name: "Val Lorentz"
    period: "2021"
    email: "progval+ircv3@progval.net"

---

## Intro

The name of this client capability MUST be named `draft/named-modes`.

This capability enables receiving PROP messages instead of MODE messages as well as introducing the `RPL_CHMODELIST` and `RPL_UMODELIST` numerics to replace the user and channel mode parameters of the `RPL_MYINFO` numeric.

## Motivation

The IRC protocol originally had a `MODE` command to configure channels, using single character as configuration keys.
Implementation extend the base protocol with their own modes, causing some of them to exhaust the namespace available. Additionally, independent implementations can accidentally implement the same feature with different characters, or use the same characters for different features.

This specification aims to resolve both these issues, by introducing a virtually infinite namespace for new configuration keys, and by offering a way for implementations to define their own sub-namespaces via vendor prefixes to avoid clashes.

## New numerics on connection

These numerics MAY occur more than once. If the reply consists of multiple lines (due to IRC length limitations) all but the last numeric MUST have a parameter containing only an asterisk (`*`) preceding the mode list.

### RPL_CHMODELIST

    :<server name> XXX <nick> [*] {<type>:<modename>[=<letter>]}+

where `<type>` is one of the following that tells the client about the nature
of the mode:

* 1 - mode is a list mode, requires a parameter when setting and unsetting (group 1 in 005 CHANMODES).
* 2 - mode is a parameter mode, requires a parameter when setting and unsetting (group 2 in 005 CHANMODES).
* 3 - mode is a parameter mode, requires a parameter when setting, requires no parameter when unsetting (group 3 in 005 CHANMODES).
* 4 - mode is a flag, i.e. no parameter (group 4 in 005 CHANMODES).
* 5 - mode is a prefix mode, requires a parameter when setting and unsetting, target is always a user on the channel.

and `<letter>` is an equivalent mode name for the `MODE` command.

Clients MUST ignore unknown types, even with multiple digits.

### RPL_UMODELIST

    :<server name> YYY <nick> [*] {<type>:<modename>[=<letter>]}+

where type is
* 3 - mode requires a parameter when setting, requires no parameter when unsetting.
* 4 - mode is a flag, it never has a parameter.

and `<letter>` is an equivalent mode name for the `MODE` command.
Clients MUST ignore unknown types, even with multiple digits.

The list given by these two numerics are in two separate namespaces; it is
possible to have the same mode name for a user mode and a channel mode,
because the target of a mode change is always unambiguous.

When this capability is negotiated, the mode lists in RPL_MYINFO
(004) MUST be considered undefined, server implementations MAY
omit the last 3 parameters for that numeric entirely and clients MUST
handle this.
Servers MAY send no RPL_MYINFO numeric at all, clients
MUST handle this case as well.

### Examples

Example 1: client connects and requests the named-modes capability

    Client: CAP LS
    Server: NICK tester
    Client: USER user 0 0 :gecos
    Server: :server.example CAP * LS :named-modes
    Client: CAP REQ named-modes
    Client: CAP END
    Server: :server.example CAP tester ACK :named-modes
    Server: :server.example 001 tester :Welcome to an IRC network, nick!user@example.com!
    Server: :server.example 002 tester :Your host is server.example, running version 0.0
    Server: :server.example 003 tester :This server was created ...
    Server: :server.example 004 tester :server.example
    Server: :server.example 005 tester :EXCEPTS=e NICKLEN=30 INVEX=I MAP MODES=4 NETWORK=Example
    Server: :server.example XXX tester 5:op=o 5:voice=v 4:private=p 4:secret=s 4:inviteonly=i 4:topiclock=t 4:noextmsg=n 4:moderated=m 3:limit=l 1:ban=b 2:key=k
    Server: :server.example YYY tester 4:oper=o 4:invisible=i 1:snomask=s 4:wallops=w

Example 2: the last line is equivalent to

    Server: :server.example YYY tester * 4:oper=o
    Server: :server.example YYY tester * 4:invisible=i
    Server: :server.example YYY tester * 1:snomask=s
    Server: :server.example YYY tester 4:wallops=w

Example 3: like any IRC message, the colon is optional when the trailing parameter contains no space and does not start with an other colon, so it is also equivalent to:

    Server: :server.example YYY tester 4:oper=o 4:invisible=i 1:snomask=s :4:wallops=w

## Listing modes on a channel

Query syntax:

    PROP <channel>

Reply syntax:

    RPL_PROPLIST <nick> <channel> :[<modename>[=<param>]] [<modename 2>[=<param 2>]] ... [<modename n>[=<param n>]]]
    RPL_ENDOFPROPLIST <nick> <channel> <modename> :End of list

Servers MAY send multiple `RPL_PROPLIST` replies before `RPL_ENDOFPROPLIST`.

Servers SHOULD not include modes of type 1 (list) or 5 (prefix) in this list as there may be too many; just like with the `MODE` command.

Servers MAY omit the `<param>` for any mode, and they SHOULD NOT add one for mode of type 4 (flag).
Clients MUST gracefully handle unexpected parameters, and SHOULD ignore them.

Example:

    Client: PROP #example
    Server: :server.example 961 modernclient #example :topiclock noextmsg limit=5
    Server: :server.example 960 modernclient #example :End of mode list

## Getting the list of a listmode on a channel

Query syntax:

    PROP <channel> <modename>

Reply syntax:

    RPL_LISTPROPLIST <nick> <channel> <modename> <mask> [<setter> <settime>]
    RPL_ENDOFLISTPROPLIST <nick> <channel> <modename> :End of list

Servers MAY send multiple `RPL_PROPLIST` replies before `RPL_ENDOFPROPLIST`.

The `settime` argument, if present, MUST be a UNIX timestamp of the time this mode
was placed.
The `setter` argument, if present, SHOULD be the nickname, `nick!user@host` mask,
or server name of the entity who placed this mode.

Example without the optional arguments:

    Client: PROP #chat :ban
    Server: :server.example BBB tester #chat ban :*!*@example.org
    Server: :server.example BBB tester #chat ban :another!banned@user.example.com
    Server: :server.example AAA tester #chat ban :End of list

Example with the optional arguments:

    Client: PROP #opers :ban
    Server: :server.example BBB tester #chat ban *!*example.org mike!mike@localhost :567890123
    Server: :server.example BBB tester #chat ban *!*@192.0.2.69 ChanServ!ChanServ@services.example.com :123123123
    Server: :server.example BBB tester #chat ban *!*@192.0.2.70 ChanServ!ChanServ@services.example.com :123123123
    Server: :server.example AAA tester #chat ban :End of list

Note that the first example is equivalent to:

    Client: PROP #chat ban
    Server: :server.example BBB tester #chat ban *!*@example.org
    Server: :server.example BBB tester #chat ban another!banned@user.example.com
    Server: :server.example AAA tester #chat ban :End of list


## Changing modes

A "mode change" means a `PROP` command that has a target and one or more
mode names and their associated parameters (if required by the mode).

An "item in the mode change" is defined as one mode and its parameter (if any)
in a mode change. A mode change MUST have at least one item in it.

A mode change from the client to the server is always a request; it being sent
does not guarantee the server will honor it, even if the client (thinks) it has
all the necessary privileges, etc.

A mode change from the server to the client is a notification about a change
which had already happened on the server by the time the client receives the
command.

Query syntax:

    PROP <target> {<+|-><modename>[=<parameter>]}+

Add (+) or remove (-) the mode called `<modename>`, using `<parameter>` as
the parameter, if required for the mode.

Client implementations MUST NOT send parameters for modes that do not require
one.
Server implementations MUST remove the meaningless parameter (and the
accompanying `=` sign) before propagating the mode change to other clients.

A mode change is a "no-op mode change" if after being processed the server,
the server ends up not changing anything.

If the mode change is not a no-op mode change it MUST be echoed back to the
client doing the change after being processed.
Servers SHOULD normally send the change to all other clients if the target
is a channel, in a similiar way to how they behave with a RFC 1459 `MODE`
command.

If a server implementation ecounters an item in the mode change with a
parameter which is invalid according to it, it is allowed to change (fix)
the parameter instead of rejecting that item in the mode change.
Clients MUST be prepared to handle a mode change echoed back to them
which has an item whose parameter is slightly or entirely different than
what was requested by the client.

A mode change from the client to the server SHOULD have at most MAXMODES
(from RPL_ISUPPORT) items. The surplus items SHOULD be treated the same way
by the server as it treats surplus modes in a RFC 1459 `MODE` command.
Usually this means that the surplus items are ignored silently and the `PROP`
and `MODE` messages generated by this mode change won't contain the surplus
items.
This behavior not required, meaning servers can choose to process
more items if they wish, for example, if the mode change comes from a
priviliged client the server is free to be more lax and process all items.

A mode change from the server to the client MAY contain an arbitrary
number of items. However it still MUST honor other limitations imposed
by the protocol such as the line length limit.
Note that this is different than how the traditional `MODE` command behaves,
which usually only contains up to MAXMODES changed modes.

## Translation between `PROP` and `MODE`

When a client send a `PROP` message that should be relayed to other clients,
servers MAY translate it to a `MODE` command using an equivalent
mode letter (as defined in `RPL_CHMODELIST`) to clients that did not
negotiate this capability, but MUST NOT send them a `PROP`.

When a client sends either `PROP` or `MODE`, servers SHOULD send them as
a `PROP` to clients who negotiated this capability.
Clients MUST accept `MODE` messages.

### Examples

*This section is not normative*

Changing channel modes

    Client: PROP #egypt +key=pyramids -topiclock +ban=*!*@example.com +ban=example!*@*

If the mode change is successful the following (or an equivalent) is sent to
all clients supporting this capability:

    Server: :nick!user@host PROP #egypt key=pyramids -topiclock +ban=*!*@example.com +ban=example!*@*

Clients not supporting this capability receive the following (or an
equivalent) mode change:

    Server: :nick!user@host MODE #egypt +kbb-t pyramids *!*@example.com example!*@*

The first two messages are, as usual, equivalent to:

    Client: PROP #egypt +key=pyramids -topiclock +ban=*!*@example.com :+ban=example!*@*
    Server: :nick!user@host PROP #egypt :key=pyramids -topiclock +ban=*!*@example.com :+ban=example!*@*


## Mode names

### Rules for choosing mode names

`<modename>` in the grammars above is defined as follows:

    <modename>        ::= [ <vendor> '/' ] <key_name>
    <unqual_modename> ::= <non-empty sequence of ascii letters, digits, hyphens ('-')>
    <vendor>          ::= <host>

They follow [the same rules as message tag names](../extensions/message-tags.html#rules-for-naming-message-tags).
The following sections define names for existing standard modes.

### Channel modes

From RFC 1459:

| Letter | Name         | Type       | Notes |
| ------ | ------------ | ---------- | ----- |
|  `o`   | `op`         | 5 (prefix) |       |
|  `v`   | `voice`      | 5 (prefix) |       |
|  `b`   | `ban`        | 1 (list)   |       |
|  `i`   | `inviteonly` | 4 (flag)   |       |
|  `l`   | `limit`      | 3 (param)  |       |
|  `m`   | `moderated`  | 4 (flag)   |       |
|  `n`   | `noextmsg`   | 4 (flag)   |       |
|  `k`   | `key`        | 2 or 3 (param) |   |
|  `p`   | `private`    | 4 (flag)   |       |
|  `s`   | `secret`     | 4 (flag)   |       |

From RFC 2812:

| Letter | Name         | Type       | Notes |
| ------ | ------------ | ---------- | ----- |
| `e`    | `banex`      | 1 (list)   | Ban exception |
| `I`    | `invex`      | 1 (list)   | invitation exception (aka. invitation mask) |

In widespread use across implementations:

| Typical letter(s) | Name          | Type       | Definition |
| ----------------- | ------------- | ---------- | ---------- |
| `a`               | `admin`       | 5 (prefix) | An implementation-defined power level, that usually cannot be kicked by ops and may or may not have power over them. Also known as "protected". |
| `h`               | `halfop`      | 5 (prefix) | A power level between voice and op, with implementation-defined privileges |
| `C`               | `noctcp`      | 4 (flag)   | Blocks CTCP messages other than ACTION |
| `q`               | `owner`       | 5 (prefix) | A power level above admin and op. Also known as "founder" |
| `P`               | `permanent`   | 4 (flag)   | Channel does not disappear when empty. |
| `R` or `r`        | `regonly`     | 4 (flag)   | Prevents users from joining unless they are authenticated with a network account. |
| `z` or `S`        | `secureonly`  | 4 (flag)   | Prevents users connected through insecure connections from joining. Also known as "sslonly" or "TLS-only". |

New modes:

| Name          | Type      | Definition | Notes |
| ------------- | --------- | ---------- | ----- |
| `mute`        | 1 (list)  | Like `ban`, but allows users to join but not to sending messages. Other privileges are implementation-defined. | aka quiet. Often implemented as an extban instead of a first-class mode. |

### User modes

From RFC 1459:

| Letter | Name         | Type       |
| ------ | ------------ | ---------- |
|  `i`   | `invisible`  | 4 (flag)   |
|  `o`   | `oper`       | 4 (flag)   |
|  `s`   | `snomask`    | 3 (param)  |
|  `w`   | `wallops`    | 4 (flag)   |

In widespread use across implementations:

| Typical letter(s) | Name         | Type       | Definition |
| ----------------- | ------------ | ---------- | ---------- |
| `B`               | `bot`        | 4 (flag)   | Marks the user as [a bot](../extensions/bot-mode.html)
| `l` or `p`        | `hidechans`  | 4 (flag)   | Hides the channel list from WHOIS. |
| `x`               | `cloak`      | 4 (flag)   | Gives the user a hidden/cloaked hostname |


### Numerics

This spec defines the following new numerics:

| No. | Label                   | Notes (will be removed before merging) |
| --- | ----------------------- | ------------ |
| 960 | `RPL_ENDOFPROPLIST`     | Used by Insp |
| 961 | `RPL_PROPLIST`          | Used by Insp |
| AAA | `RPL_ENDOFLISTPROPLIST` | Temporarily set to 962 |
| BBB | `RPL_LISTPROPLIST`      | Temporarily set to 963 |
| XXX | `RPL_CHMODELIST`        | Temporarily set to 964 |
| YYY | `RPL_UMODELIST`         | Temporarily set to 965 |

### RPL_ISUPPORT Token

This specification defines the REQUIRED `MAXMODES` token for use in `RPL_ISUPPORT` (005) responses.
This token MUST have an integer value greater than 0.

Servers should use this token to communicate to clients the maximum numbers of modes changes in a single PROP request. In the absence of this token or its value, they should assume it is 1.

Clients MUST support receiving PROP updates with more changes than the value of `MAXMODES`.
