---
title: "`server-time` Extension"
layout: spec
redirect_from:
  - /specs/extensions/server-time-3.2.html
copyrights:
  -
    name: "Stéphan Kochen"
    period: "2012"
    email: "stephan@kochen.nl"
  -
    name: "Alexey Sokolov"
    period: "2012"
    email: "alexey-irc@asokolov.org"
  -
    name: "Kyle Fuller"
    period: "2012"
    email: "inbox@kylefuller.co.uk"
  -
    name: "James Wheare"
    period: "2016"
    email: "james@irccloud.com"
---
Clients indicate support for the extension by requesting a capability server-time as per the [IRC Client Capabilities Extension][cap].

	CAP REQ :server-time

When enabled, the server-time extension enables optional `time` [message tag][] which can be used in messages from server to client.
The value of the tag MUST have the following syntax:

	<value> ::= YYYY-MM-DDThh:mm:ss.sssZ

It represents a calendar date and time of day in UTC using extended format, as specified by ISO 8601:2004(E) 4.3.2.
Precision of the time is one millisecond.

If the `time` is presented in a message received from a server, a client SHOULD treat the message as having occurred at the given time instead of its current time.

Servers MAY include the timestamp in messages when they see fit but MUST NOT do so before acknowledging the client capability using `CAP ACK`.
Clients MAY choose to simplify parsing by accepting timestamps at any point in the connection (e.g. even before `CAP REQ`).

Servers offering this capability for all messages SHOULD also offer the `echo-message` capability to allow clients to keep a consistent chronology of events.

## Examples

This section is non-normative

Angel messaged Wiz at Wed, 19 Oct 2011 16:40:51.620 UTC

	@time=2011-10-19T16:40:51.620Z :Angel!angel@example.org PRIVMSG Wiz :Hello

John joined #chan during the leap second of June 2012 (notice the :60)

	@time=2012-06-30T23:59:60.419Z :John!~john@1.2.3.4 JOIN #chan


[cap]: ../core/capability-negotiation.html
[message tag]: ../extensions/message-tags.html


## Errata

Previous versions of the specification didn't recommend `echo-message`
