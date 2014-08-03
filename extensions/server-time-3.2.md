server-time client capability specification
===========================================

Copyright (c) 2012 Stéphan Kochen <stephan@kochen.nl>  
Copyright (c) 2012 Alexey Sokolov <alexey-irc@asokolov.org>  
Copyright (c) 2012 Kyle Fuller <inbox@kylefuller.co.uk>

Clients indicate support for the extension by requesting a capability server-time as per the [IRC Client Capabilities Extension][cap].

	CAP REQ :server-time

When enabled, the server-time extension enables optional `time` [message tag][] which can be used in messages from server to client.
The value of the tag MUST have the following syntax:

	<value> ::= YYYY-MM-DDThh:mm:ss.sssZ

It represents a calendar date and time of day in UTC using extended format, as specified by ISO 8601:2004(E) 4.3.2.
Precision of the time is one millisecond.

If the `time` is presented in a message received from server, client SHOULD treat the message as the one happened at the given time instead of now.

Servers MAY include the timestamp in messages when they see fit (in order to tell the client that the message really happened at the given time instead of now),
but MUST NOT do so before acknowledging the client capability using `CAP ACK`.
Clients MAY choose to simplify parsing by accepting timestamps at any point in the connection (e.g. even before `CAP REQ`).

Example 1:

Angel messaged Wiz at Wed, 19 Oct 2011 16:40:51.620 UTC

	@time=2011-10-19T16:40:51.620Z :Angel!angel@example.org PRIVMSG Wiz :Hello

Example 2:

John joined #chan during the leap second of June 2012 (notice the :60)

	@time=2012-06-30T23:59:60.419Z :John!~john@1.2.3.4 JOIN #chan

[cap]: http://ircv3.atheme.org/specification/capability-negotiation-3.1
[message tag]: http://ircv3.atheme.org/specification/message-tags-3.2
