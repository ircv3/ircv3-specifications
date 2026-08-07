---
title: "Channel Flood Limits"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Valware"
    period: "2026"
    email: "v.a.pond@outlook.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `channel-flood-limits` capability name. Instead, implementations
SHOULD use the `draft/channel-flood-limits` capability name to be interoperable
with other software implementing a compatible work-in-progress version.

The `CHANLIMITS` command name is used in both draft and final versions of this
specification.

The final version of the specification will use an unprefixed capability name.

## Introduction

This specification adds a mechanism for servers to advertise per-channel flood
protection limits to clients, and for privileged clients (bots, services) to
set additional limits on channels they operate. With this information, clients
can rate-limit user input before it triggers server-side enforcement like kicks,
bans, or mutes, preventing users from unknowingly getting themselves actioned.

Right now, channel flood protection is configured server-side through
implementation-specific mechanisms (channel modes, services, server config) that
are not standardized and not visible to clients. Clients have no way of knowing
what the flood thresholds of a channel are, and so they can't prevent the user
from exceeding them. Bots that enforce their own flood rules have no way to
communicate those rules to the server or to other clients.

## Motivation

IRC servers commonly implement channel flood protection through proprietary
channel modes and configuration, for example:

- UnrealIRCd uses channel modes `+f` and `+F` with a complex parameter syntax
  (e.g., `+f [30j,40m,7c#C15,10n#N15]:15`) that encodes multiple flood types,
  limits, actions, and durations.
- InspIRCd uses a `+f` mode with a different syntax.
- Other servers use services bots or custom modules.

These all share the same core concepts (rate limits per time window for various
event types) but differ in syntax, features, and visibility. The problems:

1. **Clients can't discover flood limits.** There is no standard way for a
   client to know what the flood thresholds are for a given channel.
2. **Profile-based modes hide parameters.** Some implementations use named
   profiles (e.g., UnrealIRCd's `+F normal`) where the actual numeric limits
   aren't visible in the channel mode string at all.
3. **Server-wide defaults may apply invisibly.** Servers may apply default flood
   protection to all channels even when no explicit mode is set.

Without this information, clients can't do things like:

- Throttle outgoing messages to stay within per-user limits
- Warn users who are about to send duplicate/repeated content
- Show the channel's flood policy anywhere in the UI
- Prevent users from getting kicked/banned/muted for accidental flooding

Additionally, bots and services that enforce their own channel rules (e.g.,
stricter limits for new or unidentified users) have no standard way to
communicate those rules back to the server or to other clients. Each bot is a
black box. This spec gives them a way to register their limits with the server
so that all clients see one merged result.

This specification provides a standard way for servers to signal channel flood
limits to clients, and for privileged clients to set them, regardless of the
underlying implementation.

## Architecture

### Dependencies

This specification uses the [capability negotiation](https://ircv3.net/specs/extensions/capability-negotiation),
[batch](https://ircv3.net/specs/extensions/batch), and
[standard replies](https://ircv3.net/specs/extensions/standard-replies)
frameworks.

### Capability

This specification adds the `draft/channel-flood-limits` capability.

Clients requesting this capability indicate that they can handle the
`CHANLIMITS` message described below.

Servers advertising this capability will send `CHANLIMITS` messages to clients
who have negotiated it.

The capability has no value.

### The `CHANLIMITS` message

`CHANLIMITS` is used in both directions: server-to-client and client-to-server.

#### Server to client

The server sends `CHANLIMITS` to tell a client the current effective flood
protection limits for a channel. It is used both for initial delivery (on JOIN)
and for updates (when limits change).

```
CHANLIMITS <channel> <limit-param> [<limit-param> ...]
```

Or, when no flood limits apply:

```
CHANLIMITS <channel> *
```

##### Network-wide defaults

The server MAY send a `CHANLIMITS` with `*` as the channel name to advertise
network-wide default flood limits:

```
CHANLIMITS * <limit-param> [<limit-param> ...]
```

This tells the client what the baseline flood limits are for any channel on the
network, before any per-channel overrides. If sent, this SHOULD be sent during
connection registration, after `RPL_WELCOME` (`001`).

When a client joins a channel, the per-channel `CHANLIMITS` it receives
replaces the network defaults for that channel. If no per-channel `CHANLIMITS`
is sent on JOIN, the network defaults (if any) still apply.

##### Per-channel delivery

The server MUST send `CHANLIMITS` in the following cases:

1. **On JOIN**: After a client joins a channel, the server MUST send
   `CHANLIMITS` to that client with the flood limits that apply to them. The
   message can arrive at any point after the JOIN. Clients MUST NOT assume any
   particular ordering relative to `RPL_TOPIC` or `RPL_ENDOFNAMES`.

2. **On limit change**: When the flood limits for a channel change (e.g., an
   op changes the flood protection settings), the server MUST send an updated
   `CHANLIMITS` to all channel members who have negotiated the
   `draft/channel-flood-limits` capability.

3. **On user status change**: When a user's effective flood limits change
   because of a status change (being opped, deopped, identified to services,
   or having a flood exemption added/removed), the server MUST send an
   updated `CHANLIMITS` to that user for the affected channel(s).

The server MUST NOT send `CHANLIMITS` to clients that have not negotiated the
`draft/channel-flood-limits` capability.

If a channel has no flood limits (flood protection is off), or if the user is
exempt from flood limits (e.g., they are a channel operator), the server SHOULD
send `CHANLIMITS <channel> *` to indicate no limits apply. The server MAY also
omit `CHANLIMITS` entirely on JOIN to indicate no limits apply; however, the
message MUST still be sent if limits that previously applied are removed, so the
client knows to stop enforcing them.

When the effective limits for a channel would be too long for a single IRC
message, the server MUST use the `chanlimits` batch type (see
[Batch type](#batch-type)) to send them across multiple lines.

#### Client to server

Privileged clients (bots, services, or operators) can set additional flood
limits on a channel by sending `CHANLIMITS` to the server:

```
CHANLIMITS <channel> <limit-param> [<limit-param> ...]
```

The client MUST have sufficient channel privileges to set limits. What
constitutes "sufficient privileges" is left to the server implementation (it
might require +o, +a, +q, or oper status). If the client lacks privileges, the
server MUST reply with a `FAIL` message:

```
FAIL CHANLIMITS CHANOP_NEEDED <channel> :You need channel privileges to set flood limits
```

When a client sets limits, the server records them as belonging to that client.
How the server tracks ownership internally is an implementation detail (account
name, UID, module data, etc.). Multiple sources (the server's own mode-based
limits, plus limits from one or more privileged clients) may coexist on a
channel.

To remove specific limits previously set, a client sends:

```
CHANLIMITS <channel> -<scope>:<type>[,-<scope>:<type>,...]
```

For example:

```
CHANLIMITS #chat -user:message,-user:repeat
```

This removes the `user:message` and `user:repeat` limits that *this specific
client* previously set. The server then recomputes the effective limits and
broadcasts the result.

To remove all limits a client previously set on a channel at once:

```
CHANLIMITS <channel> *
```

When sent client-to-server, `*` means "clear all limits I set." The server
MUST NOT interpret this as clearing limits from other sources.

After any client-to-server `CHANLIMITS` (set or remove), the server MUST
recompute the effective limits and send an updated server-to-client `CHANLIMITS`
to all channel members who negotiated the capability.

If the client sends a `CHANLIMITS` that is syntactically invalid, the server
MUST reply with:

```
FAIL CHANLIMITS INVALID_PARAMS <channel> :Invalid flood limit syntax
```

### Multi-source merging

A channel's effective flood limits are the result of merging limits from
multiple sources:

- The server's own limits (from channel modes like `+f`/`+F`, server config,
  default profiles, etc.)
- Limits set by one or more privileged clients via `CHANLIMITS`

When merging, the server MUST pick the **strictest** limit for each scope+type
combination across all sources, meaning the lowest count for a given duration,
or the shortest duration for a given count. If two sources define the same
scope and type with different windows (e.g., `5/15` from source A and `8/30`
from source B), the server should pick whichever lets fewer events through.
Dividing count by duration gives you a rate; the lower rate wins.

The server MUST NOT expose which source contributed which limit. Clients just
see the final merged limits.

When a source is removed (e.g., a bot parts the channel, or a client sends
`CHANLIMITS #chan *` to clear its limits), the server MUST recompute the
effective limits from the remaining sources and broadcast the update.

#### Limit parameters

Each `<limit-param>` is a positional parameter of `CHANLIMITS`, formatted as:

```
<scope>:<type>=<count>/<duration>[,<type>=<count>/<duration>,...]
```

Where:

- `<scope>` identifies whether the limit applies to the channel as a whole or
  to individual users (see [Scopes](#scopes) below).
- `<type>` identifies the kind of event being limited (see [Limit types](#limit-types)
  below).
- `<count>` is a positive integer: the maximum number of events allowed within
  the window.
- `<duration>` is a positive integer: the time window in seconds.

Multiple limit types within the same scope are separated by commas (`,`).
Multiple scopes are sent as separate space-delimited parameters of the
`CHANLIMITS` message.

Each scope MUST appear at most once in a `CHANLIMITS` message. Each limit type
MUST appear at most once within a given scope. Clients SHOULD treat the last
occurrence as authoritative if duplicates are encountered.

#### Scopes

Scopes tell you whether a limit applies to the channel as a whole or to
individual users.

##### Server-to-client scopes

The server sends these scopes to clients. They are the only scopes that appear
in server-to-client `CHANLIMITS` messages.

| Scope     | Description |
|-----------|-------------|
| `channel` | An aggregate limit across all users in the channel. When the combined actions of all channel members exceed this limit within the time window, the server takes channel-wide action (e.g., setting +m). Individual users can't directly control whether this gets triggered. |
| `user`    | A per-user limit. When a single user's actions exceed this limit, the server takes action against them (e.g., kicking them). Clients can enforce these limits directly by throttling user input. |

The server MUST send each client the `user` limits that actually apply to
them. If a user is unvoiced, new to the channel, or unidentified, and the
channel has stricter limits for those conditions, the server sends that user
the stricter values under the `user` scope directly. The client doesn't need
to know *why* its limits are what they are, just what they are.

When a user's status changes (they get voiced, they identify to services, they
pass the "new user" time threshold), the server sends them updated `user`
limits reflecting their new status. This is already required by the "on user
status change" rule above.

##### Client-to-server scopes (conditional)

Privileged clients (bots, services) use these scopes when *setting* limits via
client-to-server `CHANLIMITS`. They tell the server which users a limit should
target. The server resolves them per-user and folds the result into the `user`
scope it sends to each client.

| Scope                | Description |
|----------------------|-------------|
| `user`               | Applies to all users. |
| `user-unvoiced`      | Applies to users without voice (+v) or any higher channel status (+h/+o/+a/+q). |
| `user-unidentified`  | Applies to users not identified to a services account. |
| `user-new=<seconds>` | Applies to users who joined the channel within the last `<seconds>` seconds. For example, `user-new=900` targets users who joined less than 15 minutes ago. |

When multiple conditional scopes match a user (e.g., they're both unvoiced and
new), the server picks the strictest limit for each type and sends the result
as plain `user` limits.

Clients MUST ignore scopes they do not recognize.

Vendor-specific scopes MAY be defined using the `<vendor>/<scope>` naming
convention. For example, `example.com/user-trusted` could represent a
custom user classification.

#### Limit types

The following limit types are defined:

| Type      | Applicable scopes | Description |
|-----------|-------------------|-------------|
| `message` | `channel`, `user` | PRIVMSG and NOTICE messages sent to the channel. |
| `ctcp`    | `channel`, `user` | CTCP messages (PRIVMSG with `\x01` delimiters) sent to the channel. |
| `join`    | `channel`         | JOIN events (users joining the channel). |
| `nick`    | `channel`, `user` | Nick changes by users in the channel. |
| `knock`   | `channel`         | KNOCK requests to the channel. |
| `repeat`  | `user`            | Identical consecutive messages sent by a single user to the channel. What the server considers "identical" depends on its own matching criteria, which may include case-insensitive comparison, stripping formatting, or other normalization. |

Clients MUST ignore limit types they do not recognize.

Server implementations MAY advertise a subset of these limit types depending on
what flood protections they support. Server implementations MAY define
vendor-specific limit types using the `<vendor>/<type>` naming convention,
following the same rules as [vendor-specific message tag names](https://ircv3.net/specs/extensions/message-tags#vendor-specific).

#### Window semantics

The `<count>/<duration>` rate describes a sliding time window: at any point in
time, the server counts events within the last `<duration>` seconds and triggers
enforcement when `<count>` is exceeded.

Server implementations MAY use different windowing strategies internally (fixed
windows, token buckets, etc.), but MUST advertise limits in the
`<count>/<duration>` format. The advertised values SHOULD be a conservative
approximation of the server's actual enforcement thresholds. Put another way: a
client that stays within the advertised limits SHOULD NOT trigger enforcement.

Clients SHOULD use the advertised limits as a guideline for rate-limiting user
actions. Since the server's internal windowing may differ from the client's
tracking, clients SHOULD apply a safety margin (e.g., staying 10-20% below the
advertised limit) to avoid tripping enforcement on edge cases.

### Batch type

This specification adds the `chanlimits` batch type for use with the
[batch](https://ircv3.net/specs/extensions/batch) extension.

When the effective limits for a channel have many limit types and would exceed
the 512-byte IRC message limit, the server MUST use a batch to deliver them:

```
:irc.example.com BATCH +ref chanlimits #channel
@batch=ref :irc.example.com CHANLIMITS #channel channel:message=40/15,join=30/15,nick=8/15,ctcp=7/15,knock=10/15
@batch=ref :irc.example.com CHANLIMITS #channel user:message=5/15,repeat=3/15
:irc.example.com BATCH -ref
```

Within a `chanlimits` batch, each `CHANLIMITS` line carries one or more scopes
for the same channel. The batch as a whole replaces all previously known limits
for that channel. Clients MUST NOT merge batch contents with previously cached
limits; the batch is a complete snapshot.

Servers MAY use this batch type even when limits would fit in a single line,
for consistency.

Servers MAY also use this batch type on JOIN to deliver limits alongside other
channel state, in which case the batch SHOULD appear in the same position as a
non-batched `CHANLIMITS` (after topic, before NAMES).

### Interaction with other specifications

#### Capability negotiation

The `draft/channel-flood-limits` capability does not depend on any other
capability, but works well alongside:

- [batch](https://ircv3.net/specs/extensions/batch): Required for delivering
  limits that exceed a single IRC line. Servers MUST support batch if they
  advertise `draft/channel-flood-limits`.
- [echo-message](https://ircv3.net/specs/extensions/echo-message): Clients
  using echo-message to confirm sent messages can use flood limits to avoid
  sending messages that would be silently dropped or get them actioned.
- [draft/multiline](https://ircv3.net/specs/extensions/multiline): Each line
  within a multiline batch is typically counted individually toward message
  flood limits. Clients with multiline support should take channel flood limits
  into account when constructing batches.

#### Channel modes

This specification does not try to standardize the underlying channel modes or
configuration that controls flood limits. Different servers use different modes
(`+f`, `+F`, custom modules, services, etc.) and this spec sits as an
abstraction layer over all of them.

Servers SHOULD send updated `CHANLIMITS` when the effective flood limits change,
no matter what caused the change (mode change, services command, config reload,
etc.).

#### Extended bans and exemptions

Some servers allow per-user flood exemptions through extended ban types (e.g.,
UnrealIRCd's `~flood` extended ban). When a user has such an exemption, the
server SHOULD reflect this in the `CHANLIMITS` sent to that user, either by
increasing the limits or sending `*`.

## Client implementation considerations

This section is non-normative.

### What clients need to know

Because the server sends each user their own resolved limits, clients don't
need to know about conditional scopes at all. The server has already figured
out which limits apply to you and sends the result as plain `channel` and
`user` scopes. When your status changes (you get voiced, you identify, you've
been in the channel long enough), the server sends you updated limits.

All a client needs to do is enforce the `user` limits it's been given.

### Rate limiting user input

The main use case here is letting clients throttle outgoing messages so users
don't get penalized for flooding.

For `user`-scoped `message` limits, clients can keep a local token bucket or
sliding window to track the user's recent messages. When the user tries to send
a message that would exceed the limit, the client can:

1. **Delay sending** until the rate allows it, with a visual indicator showing
   a brief cooldown.
2. **Warn the user** before sending, letting them decide whether to proceed.
3. **Show the limit** in the channel UI so the user knows what the constraint
   is.

Clients SHOULD prefer delaying messages over rejecting them outright.

### Repeat detection

For `user`-scoped `repeat` limits, clients can track recently sent messages and
warn users who try to send identical (or near-identical) content. Since the
server's exact matching criteria aren't known to the client, a simple
exact-match comparison (possibly case-insensitive) is a reasonable starting
point.

### Channel-wide limits

`channel`-scoped limits are less directly actionable since a single user
doesn't control channel-wide traffic. But clients MAY still:

- Show the channel's flood policy as informational metadata.
- Warn when the channel seems to be approaching a channel-wide limit (e.g.,
  if the client can see a high message rate in incoming traffic).
- Use channel-wide limits as an extra signal to throttle the user's own
  messages during busy periods.

### Exempt users

When a client receives `CHANLIMITS <channel> *`, it SHOULD remove any flood
rate-limiting for that channel. This typically happens when the user gets opped
or otherwise exempted from flood protection.

### Handling unknown types and scopes

Clients MUST silently ignore limit types and scopes they don't understand, so
the spec can be extended in the future. Clients SHOULD still process and apply
the types they do recognize from the same message.

## Server implementation considerations

This section is non-normative.

### Mapping implementation-specific modes

Servers should translate their internal flood protection config to the limit
types and format defined here.

For example, an UnrealIRCd server with `+f [30j,40m#M10,5t#b]:15` (30 channel-wide joins,
 40 channel messages, and 5 individual user messages per 15 seconds) would send:

```
:irc.example.com CHANLIMITS #channel channel:message=40/15,join=30/15 user:message=5/15
```

The server-specific enforcement actions (`#M10` to set +M for 10 minutes, `#b` to ban
the user) and removal timers are not included. This spec only deals with the rate
thresholds, not what happens when they're exceeded.

For profile-based modes like UnrealIRCd's `+F normal`, the server should
resolve the profile to its constituent limits and advertise those:

```
:irc.example.com CHANLIMITS #channel channel:message=40/15,join=30/15,nick=8/15,ctcp=7/15,knock=10/15
```

### Default flood protection

If the server applies default flood protection even without an explicit mode
being set (e.g., UnrealIRCd 6.2.0+'s default `+F` profile), it SHOULD still be
advertised via `CHANLIMITS`.

### Per-user personalization

The server is responsible for resolving all conditional scopes and sending each
user the limits that actually apply to *them*. Clients only ever see `channel`
and `user` scopes.

For example, if a bot has set `user-new=900:message=2/15` and
`user-unvoiced:message=4/15`, and the base channel limits include
`user:message=8/15`, then:

- A user who joined 5 minutes ago and has no voice gets `user:message=2/15`
  (the strictest match).
- A user who's been around for an hour but has no voice gets
  `user:message=4/15`.
- A voiced user gets `user:message=8/15` regardless of join time.

When a user's status changes (they get voiced, they identify to services, they
pass the "new user" time threshold), the server SHOULD send an updated
`CHANLIMITS` for each channel they're in with the new effective `user` limits.

### Operator exemptions

When a user is given channel operator status (or any status that makes them
exempt from flood protection), the server SHOULD send:

```
:irc.example.com CHANLIMITS #channel *
```

And conversely, when that status is removed, the server SHOULD re-send the
applicable limits.

### Overlay semantics

Some servers allow multiple flood protection mechanisms to stack (e.g.,
UnrealIRCd's `+f` overrides specific settings from `+F`). With the addition
of client-set limits, there may now be many sources. The server SHOULD work out
the effective limits from all active sources and advertise only the final merged
values. Clients shouldn't need to understand or reconcile overlapping configs.

See [Multi-source merging](#multi-source-merging) for how the merge works.

### Client-set limits lifecycle

When a privileged client that previously set limits parts the channel, is
kicked, or disconnects, the server SHOULD remove that client's limits and
recompute the effective limits. This prevents stale limits from persisting
after the bot/service that set them is gone.

Servers MAY choose to persist client-set limits for some grace period to handle
brief disconnects, but MUST eventually clean them up.

If a client sets limits while identified to an account, and later reconnects
and re-identifies, the server MAY restore their previously set limits. This is
left to the server implementation.

### Performance

Servers should be mindful of how many `CHANLIMITS` messages they send when
limits change. If a config reload causes flood limits to change on many channels
at once, the server SHOULD batch or throttle the outgoing `CHANLIMITS` messages
to avoid flooding clients (ironic as that would be).

## Examples

This section is non-normative.

### Capability negotiation

```
Client: CAP LS 302
Server: CAP * LS :draft/channel-flood-limits

Client: CAP REQ :draft/channel-flood-limits
Server: CAP * ACK :draft/channel-flood-limits
```

### Network-wide defaults

After connection registration, the server advertises its baseline flood limits:

```
Server: :irc.example.com 001 nick :Welcome to the Example Network
...
Server: :irc.example.com CHANLIMITS * channel:message=60/15,join=45/15,nick=10/15,ctcp=7/15,knock=10/15 user:message=8/15,repeat=5/15
```

These apply to any channel the user joins unless overridden by a per-channel
`CHANLIMITS`.

### Receiving limits on JOIN

A client joins a channel with comprehensive flood protection:

```
Client: JOIN #chat
Server: :nick!user@host JOIN #chat
Server: :irc.example.com 332 nick #chat :Welcome to #chat!
Server: :irc.example.com 333 nick #chat chanop 1709942400
Server: :irc.example.com CHANLIMITS #chat channel:message=40/15,join=30/15,nick=8/15,ctcp=7/15,knock=10/15 user:message=5/15,repeat=3/15
Server: :irc.example.com 353 nick = #chat :nick @chanop +voice
Server: :irc.example.com 366 nick #chat :End of /NAMES list
```

### Receiving personalized limits on JOIN

A channel where a bot has set stricter limits for new and unvoiced users. The
user joining is new, unvoiced, and unidentified, so the server resolves all
the conditional scopes and sends the strictest matching limits as plain `user`:

```
Client: JOIN #moderated
Server: :nick!user@host JOIN #moderated
Server: :irc.example.com 332 nick #moderated :Welcome
Server: :irc.example.com CHANLIMITS #moderated channel:message=40/15,join=30/15 user:message=2/15,repeat=2/15
Server: :irc.example.com 353 nick = #moderated :nick @chanop +guard-bot
Server: :irc.example.com 366 nick #moderated :End of /NAMES list
```

The user got `user:message=2/15` because the bot's `user-new=900` scope was
the strictest match. After 15 minutes, the server will send an update with
relaxed limits.

### Receiving limits on JOIN (per-user only)

A channel with only per-user text flood protection:

```
Server: :irc.example.com CHANLIMITS #help user:message=10/30
```

### Limits change mid-session

An operator changes the flood protection settings:

```
Server: :irc.example.com CHANLIMITS #chat channel:message=60/15,join=45/15,nick=10/15,ctcp=7/15,knock=10/15 user:message=8/15,repeat=5/15
```

All channel members who negotiated `draft/channel-flood-limits` receive this
updated message.

### User becomes exempt

A user is given operator status:

```
Server: :chanop!op@host MODE #chat +o nick
Server: :irc.example.com CHANLIMITS #chat *
```

The client removes all flood rate-limiting for #chat.

### User loses exemption

The user is deopped:

```
Server: :chanop!op@host MODE #chat -o nick
Server: :irc.example.com CHANLIMITS #chat channel:message=40/15,join=30/15 user:message=5/15
```

The client re-enables flood rate-limiting for #chat.

### No flood protection

A channel with no flood protection at all:

```
Server: :irc.example.com CHANLIMITS #wild-west *
```

Or the server may simply omit the `CHANLIMITS` message on JOIN.

### Channel-wide limits only

A channel using only channel-wide aggregate flood protection (no per-user
enforcement):

```
Server: :irc.example.com CHANLIMITS #general channel:message=60/15,join=50/15
```

### User identifies to services

A user who was getting the stricter "unidentified" limits identifies to
NickServ. The server recalculates and sends the less restrictive limits that
now apply:

```
Server: :irc.example.com CHANLIMITS #chat channel:message=40/15,join=30/15 user:message=8/15,repeat=5/15
```

The client sees its `user` limits just got more generous. It doesn't need to
know why.

### Bot sets limits on a channel

A bot with operator status sets stricter limits for new users:

```
Bot: CHANLIMITS #chat user-new=900:message=2/15,repeat=2/15 user-unvoiced:message=4/15
```

The server records these as belonging to the bot, merges them with the
channel's existing limits (from +f/+F/defaults), resolves the result per user,
and sends each channel member their personalized effective limits.

### Bot removes its limits

The bot decides to relax its rules:

```
Bot: CHANLIMITS #chat -user-new=900:message,-user-new=900:repeat,-user-unvoiced:message
```

Or to remove everything it set at once:

```
Bot: CHANLIMITS #chat *
```

The server recomputes from the remaining sources and broadcasts the update.

### Privilege error

A non-opped user tries to set limits:

```
Client: CHANLIMITS #chat user:message=1/60
Server: FAIL CHANLIMITS CHANOP_NEEDED #chat :You need channel privileges to set flood limits
```

### Vendor-specific limit types

A server with a custom flood protection module that limits URL posting:

```
Server: :irc.example.com CHANLIMITS #chat user:message=5/15,example.com/url=2/60
```

The client would ignore the unrecognized `example.com/url` limit type but
still apply the `message` limit.

## Formal grammar

The following ABNF grammar defines the syntax of the `CHANLIMITS` parameters
(after the channel name).

```abnf
; Server-to-client: full limit set or "no limits"
; Client-to-server: set limits, remove limits, or clear all
;
; Note: server-to-client messages only use the "channel" and "user" scopes.
; The conditional scopes (user-unvoiced, user-unidentified, user-new) are
; only used in client-to-server messages.
chanlimits-params = nolimit / limit-set / removal-set

nolimit           = "*"

limit-set         = limit-param *( SP limit-param )

limit-param       = scope ":" limit-list

scope             = base-scope [ scope-param ]
                  / vendor-scope [ scope-param ]
base-scope        = "channel" / "user" / "user-unvoiced"
                  / "user-unidentified" / "user-new"
vendor-scope      = vendor "/" 1*scope-char
scope-param       = "=" 1*DIGIT           ; e.g., user-new=900
scope-char        = ALPHA / DIGIT / "-"

limit-list        = limit-entry *( "," limit-entry )

limit-entry       = limit-type "=" count "/" duration

limit-type        = type-name / vendor-type
type-name         = 1*( ALPHA / DIGIT / "-" )
vendor-type       = vendor "/" type-name
vendor            = 1*( ALPHA / DIGIT / "-" / "." )

count             = 1*DIGIT               ; positive integer
duration          = 1*DIGIT               ; positive integer, seconds

; Removal (client-to-server only)
removal-set       = removal-entry *( "," removal-entry )
removal-entry     = "-" scope ":" limit-type
```

## Security considerations

### Untrusted data

Clients should treat all values in `CHANLIMITS` as untrusted. In particular:

- `<count>` and `<duration>` values should be sanity-checked as reasonable
  positive integers. Extremely large or zero values should be treated as "no
  effective limit."
- Unknown scope and type names should be ignored, not cause errors.

### Client-set limit trust

Servers MUST validate that clients setting limits via `CHANLIMITS` have
sufficient channel privileges. Servers SHOULD also apply sanity limits to
client-set values (e.g., rejecting a limit of `message=1/3600` as
unreasonably restrictive, or limits where count is 0).

Servers SHOULD NOT blindly trust that a bot's limits are sensible. The server
is the final authority on what gets advertised.

### Information disclosure

Flood protection limits reveal some information about a channel's configuration
and the server's security posture. Server operators should be aware that
advertising these limits makes them visible to all channel members. In practice
this is fine, since flood protection parameters are usually already visible
through channel mode inspection where possible, and aren't considered sensitive.

### Abuse prevention

This specification helps clients stay *within* advertised limits, not circumvent
them. Malicious clients could use the advertised limits to send traffic at
exactly the maximum allowed rate. This isn't really a new concern, since such
clients can already probe limits by trial and error, but server operators should
be aware that precise limit advertisement makes this marginally easier.

## Errata

This is the initial version of this specification. Errata will be noted here
as they are discovered.
