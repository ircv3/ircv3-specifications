---
title: IRCv3 history Batch Type
layout: spec
copyrights:
  -
    name: "Daniel Oakley"
    period: "2019"
    email: "daniel@danieloaks.net"
---
Channel and private query history playback using the `chathistory` batch can
only convey the `PRIVMSG` and `NOTICE` messages that were sent. Particularly
with channel history, having available such as joins and parts, quits,
`TAGMSG` messages, and others can be very useful.

This document describes the `history` batch type which can include `PRIVMSG`,
`NOTICE`, and any other messages that the server wishes to send as part of
the history of a given chat. It also describes the `event-playback` client
capability, which enables using this batch type instead of `chathistory`.


## The `event-playback` capability
This capability indicates support for the `history` batch type (and the
`HEVENT` message described here). Clients that negotiate this capability
will receive `history` batches in place of `chathistory` batches.


## The `history` batch type
This batch type takes one parameter. This parameter contains the target of
messages within the batch. The target MUST be either the nick of the remote
client for private messages or the name of a channel which the local client
is in for public messages.

When a client which has enabled the [`batch`][batch] and `event-playback`
capabilities performs an action which requires chat history to be relayed
to it (e.g. joining a channel with InspIRCd's `chathistory` mode enabled)
the server SHOULD replay all historical messages for that target in a single
`history` batch. If the client has also enabled the
[`server-time` capability][server-time] then messages inside the batch
MUST be tagged with the time at which they were originally sent.

The `PRIVMSG` and `NOTICE` messages MUST be sent as `PRIVMSG` and `NOTICE`
messages. Any other messages which get relayed as part of history (`TAGMSG`,
`JOIN`/`PART`, `QUIT`, etc) MUST be sent as a `HEVENT` message described
below.


## The `HEVENT` message
This message type ensures that clients interpret history playback of
arbitrary messages (`JOIN`, `PART`, `QUIT`, etc) as historical messages
rather than ones that should affect the current connection state.

The format of this message is:

    :orig-source HEVENT <orig-verb> [orig-parameters]

That is, the verb of `HEVENT`. `orig-source` is the source/prefix of the
original message being replayed. `<orig-verb>` is the verb of the
original message, and `[orig-parameters]` are the parameters of the
message that's being replayed (if any). The `HEVENT` message SHOULD also
include any tags which were on the original message being replayed.

If a client is not able to interpret an incoming `HEVENT` (for example,
they do not undersand the given `<orig-verb>`), the client MUST silently
drop the message and continue.

This message MAY ONLY be sent to clients that have negotiated the
`event-playback` capability.


## Examples

### Channel Message

    :irc.host BATCH +sxtUfAeXBgNoD history #channel
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:40:31.230Z :foo!foo@example.com PRIVMSG #channel :I like turtles.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:43:53.410Z :bar!bar@example.com NOTICE #channel :Tortoises are better.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:44:23.423Z :bar!bar@example.com HEVENT PART #channel :Tortises for life!
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:48:18.140Z :irc.host PRIVMSG #channel :Squishy animals are inferior to computers.
    :irc.host BATCH -sxtUfAeXBgNoD

### Private Message

    :irc.host BATCH +sxtUfAeXBgNoD history remote
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:40:31.230Z :remote!foo@example.com PRIVMSG local :I like turtles.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:43:53.410Z :local!bar@example.com PRIVMSG remote :Tortoises are better.
    :irc.host BATCH -sxtUfAeXBgNoD

[batch]: ../batch-3.2.html
[server-time]: ../server-time-3.2.html
