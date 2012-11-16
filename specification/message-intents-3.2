# Message Intents framework

Copyright (c) 2012 William Pitcock <nenolod@dereferenced.org>.

Unlimited redistribution and modification is allowed provided that the above
copyright notice and this permission notice remains intact.

This specification concerns adding an optional out-of-band message tag to messages
using the message tagging framework.  Message intent tags should be considered an
initial step in replacing CTCP.

## Background: CTCP intent encapsulation

CTCP provides a way of encapsulating messages with intent information.  This is done
by encapsulating the message body inside 0x1 bytes, with the first word in the message
body being a keyword which identifies the intent of the message, such as "ACTION" or
"DCC".

The purpose of this specification is to obsolete this encoding method using the message
tagging framework included in IRCv3.

## The `intents` capability

A client MUST request the `intents` capability in order to receive intent tags.  IRCd
software MAY represent intent tagged messages in the legacy encapsulation format described
above for clients which did not request the `intents` capability.

## The `intent` message tag

To send an intent tagged message, such as an ACTION intent, we do the following:

	@intent=ACTION PRIVMSG #ircv3 :kicks Jobe

The IRC server would then attach the `intent` tag to the message when it distributes
it to clients supporting the `intent` tag.

## Legacy translation

The IRC server MAY translate intents to the legacy encoding and MAY translate intents
from the legacy CTCP encoding as well.

During the transition period, the IRC server SHOULD perform this translation until a
point in time is reached where translation is no longer necessary.

## Metadata vs. Intents

Some CTCP intents were used for the purpose of querying metadata.  Instead, the IRCv3
metadata framework should be used instead.  The Intents framework should be used strictly
for messages which need to be reinterpreted or active messages instead of queries.
