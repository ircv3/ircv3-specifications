---
title: IRCv3 chathistory Batch Type
layout: spec
redirect_from:
  - /specs/extensions/batch/chathistory-3.3.html
copyrights:
  -
    name: "Peter Powell"
    period: "2015"
    email: "petpow@saberuk.com"
---
## `chathistory` Batch Type

This document describes the format of the `chathistory` batch type.

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

[batch]: ../batch-3.2.html
[server-time]: ../server-time-3.2.html
