---
title: IRCv3 chathistory Batch Type
layout: spec
copyrights:
  -
    name: "Peter Powell"
    period: "2015"
    email: "petpow@saberuk.com"
  -
    name: "Daniel Oakley"
    period: "2019"
    email: "daniel@danieloaks.net"
---
## `chathistory` Batch Type

This document describes the format of the `chathistory` batch type and the
related `event-playback` capability.

The `chathistory` batch type takes one parameter. This parameter contains the
target of messages within the batch. The target MUST either be the nick of the
remote client for private messages or the name of a channel which the
local client is in for public messages.

When a client which has enabled the [`batch` capability][batch] performs an
action which requires chat history to be relayed to it (e.g. joining a channel
with InspIRCd's `chanhistory` mode enabled) the server SHOULD put all PRIVMSGs
and NOTICEs into a single `chathistory` batch. If the client has also enabled
the [`server-time` capability][server-time] then messages inside the batch MUST
be tagged with the time at which they were originally sent.

Servers MAY ONLY send `PRIVMSG` and `NOTICE` messages inside `chathistory`
batches, unless the `event-playback` capability described below is in use.

### Arbitrary Event Playback

If a client negotiates the `event-playback` capability, the server MAY send
that client any other event as part of history playback using the `CHEVENT`
(chat history event) message. The format of this message is:

    CHEVENT <orig-verb> [orig-parameters]

That is, the verb is `CHEVENT`. `<orig-verb>` is the verb of the original
message being replayed (e.g. `JOIN`). `[orig-parameters]` are the parameters of
the original message. The `CHEVENT` message SHOULD include any tags which were
on the original message that's being replayed.

The server MAY replay any message to a client using this method, if that client
has negotiated the `event-playback` capability. If the client has not negotiated
this capability and the server wishes to replay a given event, the server MAY
create a human-readable `PRIVMSG` or `NOTICE` with a prefix of of the server
name, containing the details for the user to interpret themselves.

## Examples

### Channel Message

    :irc.host BATCH +sxtUfAeXBgNoD chathistory #channel
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:40:31.230Z :foo!foo@example.com PRIVMSG #channel :I like turtles.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:43:53.410Z :bar!bar@example.com NOTICE #channel :Tortoises are better.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:48:18.140Z :irc.host PRIVMSG #channel :Squishy animals are inferior to computers.
    :irc.host BATCH -sxtUfAeXBgNoD

### Private Message

    :irc.host BATCH +sxtUfAeXBgNoD chathistory remote
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:40:31.230Z :remote!foo@example.com PRIVMSG local :I like turtles.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:43:53.410Z :local!bar@example.com PRIVMSG remote :Tortoises are better.
    :irc.host BATCH -sxtUfAeXBgNoD

### Event Replay to a client with the `event-playback` capability

    :irc.host BATCH +sxtUfAeXBgNoD chathistory #channel
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:39:12.230Z :foo!foo@example.com CHEVENT JOIN #channel
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:40:31.230Z :foo!foo@example.com PRIVMSG #channel :I like turtles.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:43:53.410Z :bar!bar@example.com NOTICE #channel :Tortoises are better.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:46:40.410Z :bar!bar@example.com CHEVENT PART #channel :Leaving channel
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:48:18.140Z :irc.host PRIVMSG #channel :Squishy animals are inferior to computers.
    :irc.host BATCH -sxtUfAeXBgNoD

### Event Replay to a client lacking the `event-playback` capability

    :irc.host BATCH +sxtUfAeXBgNoD chathistory #channel
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:39:12.230Z :irc.example.com PRIVMSG #channel :** foo joined #channel
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:40:31.230Z :foo!foo@example.com PRIVMSG #channel :I like turtles.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:43:53.410Z :bar!bar@example.com NOTICE #channel :Tortoises are better.
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:46:40.410Z :irc.example.com PRIVMSG #channel :** bar left #channel [Leaving channel]
    @batch=sxtUfAeXBgNoD;time=2015-06-26T19:48:18.140Z :irc.host PRIVMSG #channel :Squishy animals are inferior to computers.
    :irc.host BATCH -sxtUfAeXBgNoD

[batch]: ../batch-3.2.html
[server-time]: ../server-time-3.2.html
