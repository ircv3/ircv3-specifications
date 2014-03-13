self-message client capability specification
---------------------------------------------

Copyright (c) 2014 Attila Molnar <attilamolnar@hush.com>

Unlimited redistribution and modification of this document is allowed
provided that the above copyright notice and this permission notice
remains in tact.

## Description

This client capability MUST be named `self-message`.

If enabled, clients that receive `PRIVMSG` and `NOTICE` messages from the
server with the message source being themselves, they MUST treat the
message the same way as if the client itself would have sent the message
to the target.

## Use case

This is useful when a user has multiple clients attached to a bouncer.
In this scenario, when the user sends a message from one client the bouncer
can relay the message to the other attached clients, allowing all clients
to display a conversation.

## Examples

The client using this cap in this example is `example!ex@example.com`.

    Server: :example!ex@example.com PRIVMSG Attila :hi
    Server: :example!ex@example.com PRIVMSG #ircv3 :back from lunch

The client interprets the first line as if it had sent a `PRIVMSG` with
contents "hi" to `Attila` and the second message as if it had sent a
`PRIVMSG` with contents `back from lunch` to `#ircv3`.
