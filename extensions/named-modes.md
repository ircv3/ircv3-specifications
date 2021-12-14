---
title: IRCv3 Named Modes Extension
layout: spec
work-in-progress: true
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

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `named-modes` capability name. Instead, implementations SHOULD use the `draft/named-modes` capability name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use an unprefixed name.

## Introduction

This specification describes a mechanism to advertise and manipulate additional modes (channel and user configuration settings) beyond those specified in the original IRC RFCs.

## Motivation

The IRC protocol originally had a `MODE` command to configure channels, using single character as configuration keys.
Implementations extend the base protocol with their own modes, causing some of them to exhaust the namespace available. Additionally, independent implementations can accidentally implement the same feature with different characters, or use the same characters for different features.

This specification aims to resolve both these issues, by introducing a virtually infinite namespace for new configuration keys, and by offering a way for implementations to define their own sub-namespaces via vendor prefixes to avoid clashes.

## Capability

The `draft/named-modes` capability allows clients to indicate support for receiving the new commands and numerics defined in this specification. Clients MUST negotiate this capability before using this functionality.

## New numerics on connection

These numerics MAY occur more than once. If the reply consists of multiple lines (due to IRC length limitations) all but the last numeric MUST have a parameter containing only an asterisk (`*`) preceding the mode list.

### RPL_CHMODELIST

    :<server name> XXX <nick> [*] {<type>:<modename>[=<letter>]}+

where `<type>` is one of the following that tells the client about the nature
of the mode:

* 1 - mode is a list mode that requires a parameter when setting and unsetting (group 1 in 005 CHANMODES).
* 2 - mode is a parameter mode that requires a parameter when setting and unsetting (group 2 in 005 CHANMODES).
* 3 - mode is a parameter mode that requires a parameter when setting and no parameter when unsetting (group 3 in 005 CHANMODES).
* 4 - mode is a flag mode that requires no parameter (group 4 in 005 CHANMODES).
* 5 - mode is a prefix mode that requires a parameter when setting and unsetting, target is always a user on the channel (any mode from 005 PREFIX).

and `<letter>` is an equivalent mode name for the `MODE` command.

Clients MUST ignore unknown types, even with multiple digits.

### RPL_UMODELIST

    :<server name> YYY <nick> [*] {<type>:<modename>[=<letter>]}+

where type is
* 3 - mode requires a parameter when setting and no parameter when unsetting.
* 4 - mode is a flag, i.e. requires no parameter.

and `<letter>` is an equivalent mode name for the `MODE` command.
Clients MUST ignore unknown types, even with multiple digits.

The list given by these two numerics are in two separate namespaces; it is
possible to have the same mode name for a user mode and a channel mode,
because the target of a mode change is always unambiguous.

When this capability is negotiated, clients MUST consider the mode lists in
RPL_MYINFO (004) to be undefined. Server implementations MAY send undefined,
syntactically valid placeholder parameters as the final three parameters
of RPL_MYINFO.

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
    Server: :server.example 004 tester server.example ExampleIRCd-1.0.0 iosw :beIhiklmnotvPZ
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

Example 4: The IRCd provides a custom mode to control chat history availability:

    Server: :server.example XXX tester 3:example.org/history

## Listing modes on a channel

Query syntax:

    PROP <channel>

Reply syntax:

    RPL_PROPLIST <nick> <channel> [<modename>[=<param>]] [<modename 2>[=<param 2>]] ... [<modename n>[=<param n>]]]
    RPL_ENDOFPROPLIST <nick> <channel> <modename> :End of list

Servers MAY send multiple `RPL_PROPLIST` replies before `RPL_ENDOFPROPLIST`.

Servers SHOULD not include modes of type 1 (list) or 5 (prefix) in this list as there may be too many (as with the `MODE` command).

Servers MAY omit the `<param>` for any mode, and they SHOULD NOT add one for mode of type 4 (flag).
Clients MUST gracefully handle unexpected parameters, and SHOULD ignore them.

Example:

    Client: PROP #example
    Server: :server.example 961 modernclient #example topiclock noextmsg :limit=5
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

Client and server implementations MUST NOT send parameters for modes that
do not require them.

A mode change is a "no-op mode change" if after being processed the server,
the server ends up not changing anything.
If the mode change is not a no-op mode change it MUST be echoed back to the
client doing the change after being processed.
Servers SHOULD normally send the change to all other clients if the target
is a channel, in a similar way to how they behave with a RFC 1459 `MODE`
command.

If a server implementation encounters an item in the mode change with a
parameter which is invalid according to the server, it is allowed to change
(fix) the parameter instead of rejecting that item in the mode change.
Clients MUST accept mode changes echoed back to them where one or more
parameters differ from the ones they requested.

A mode change from the client to the server SHOULD have at most MAXMODES
(from RPL_ISUPPORT) items. Servers MAY silently drop mode changes beyond
the MAXMODES limit.

A mode change from the server to the client MAY contain an arbitrary
number of items (within the syntactic limits imposed by the IRC protocol).
(Note that this differs from the behavior of the `MODE` command,
which usually only contains up to MAXMODES changed modes.)

## Translation between `PROP` and `MODE`

When relaying a `PROP` message, servers MAY translate it to a `MODE` message
using an equivalent mode letter (as defined in `RPL_CHMODELIST`) for clients
that did not negotiate this capability. Similarly, servers MAY translate
`MODE` messages to `PROP` messages for clients that did negotiate the
capability.

Servers MUST NOT send `PROP` to clients that have not negotiated the
capability. Clients MUST accept `MODE` messages.

### Examples

*This section is not normative*

Example 1: Changing channel modes

    Client: PROP #egypt +key=pyramids -topiclock +ban=*!*@example.com +ban=example!*@*

If the mode change is successful the following (or an equivalent) is sent to
all clients supporting this capability:

    Server: :nick!user@host PROP #egypt key=pyramids -topiclock +ban=*!*@example.com +ban=example!*@*

Clients not supporting this capability receive the following (or an
equivalent) mode change:

    Server: :nick!user@host MODE #egypt +kbb-t pyramids *!*@example.com example!*@*

Example 2: The first two messages are, as usual, equivalent to:

    Client: PROP #egypt +key=pyramids -topiclock +ban=*!*@example.com :+ban=example!*@*
    Server: :nick!user@host PROP #egypt :key=pyramids -topiclock +ban=*!*@example.com :+ban=example!*@*

Example 2: The client changes a mode with a vendor prefix:

    Client: PROP #egypt +example.org/history=10:20m
    Server: :nick!user@host PROP #egypt :+example.org/history=10:20m


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
|  `t`   | `topiclock`  | 4 (flag)   |       |

From RFC 2812:

| Letter | Name         | Type       | Notes |
| ------ | ------------ | ---------- | ----- |
| `e`    | `banex`      | 1 (list)   | Ban exception |
| `I`    | `invex`      | 1 (list)   | invitation exception (aka. invitation mask) |

In widespread use across implementations:

| Typical letter(s) | Name          | Type       | Definition |
| ----------------- | ------------- | ---------- | ---------- |
| `a`               | `admin`       | 5 (prefix) | An implementation-defined power level, that usually cannot be kicked by ops and may or may not have power over them. Also known as "protected". |
| `h`               | `halfop`      | 5 (prefix) | A power level between voice and op, with implementation-defined privileges. |
| `C`               | `noctcp`      | 4 (flag)   | Blocks CTCP messages other than ACTION. |
| `q`               | `owner`       | 5 (prefix) | A power level above admin and op. Also known as "founder". |
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
| `B`               | `bot`        | 4 (flag)   | Marks the user as [a bot](../extensions/bot-mode.html). |
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
