---
title: IRCv3 cap-value Batch Type
layout: spec
copyrights:
  -
    name: "Alexey Sokolov"
    period: "2015"
    email: "alexey-irc@asokolov.org"
---
## `cap-value` Batch Type

Capability negotiation 3.2 introduced values for capabilities.
When server needs to change value after negotiation, it can do that using CAP
DEL followed by CAP NEW, but to make it a single transaction, server SHOULD put
them in a batch `cap-value`.

Even though it is a single transaction, client MUST use `CAP REQ` again if it
wants the capability back.

Example:

        Client: CAP LS 302
        Server: :irc.example.com CAP * LS :sasl=PLAIN batch
        Client: CAP REQ batch
        ...
        Server: :irc.example.com BATCH +cap cap-value
        Server: @batch=cap :irc.example.com CAP client DEL :sasl
        Server: @batch=cap :irc.example.com CAP client NEW :sasl=PLAIN,EXTERNAL
        Server: :irc.example.com BATCH -cap

