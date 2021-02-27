---
title: "`echo-message` Extension"
layout: spec
redirect_from:
  - /specs/extensions/echo-message-3.2.html
copyrights:
  -
    name: "Attila Molnar"
    period: "2014"
    email: "attilamolnar@hush.com"
  -
    name: "J-P Nurmi"
    period: "2014"
    email: "jpnurmi@gmail.com"
---
## Description

This client capability MUST be named `echo-message`.

If enabled, servers MUST send `PRIVMSG` and `NOTICE` messages back to
the client that sent them. If servers apply any modifications to these
messages, they MUST send the final version of the message back to the
originating client.

For clients, receiving a message with themselves as the sender acts as
an acknowledgement that the message has been delivered to the server.
Clients that receive self-sent `PRIVMSG` and `NOTICE` messages, MUST
treat them the same way as if the client itself would have sent the
message to the target. Clients may choose to disable local echoing
of sent `PRIVMSG` and `NOTICE` messages altogether, or present them
in pending state.

When the [`message-tags`](../extensions/message-tags.html) capability
has been enabled, the same rules apply to `TAGMSG` messages.

## Use cases

The capability is useful for clients to get an acknowledgement that a
message was delivered. It can be also used for measuring lag between
client and server, that is, the time it takes from sending a message
to receiving it back.

Furthermore, the capability is useful for users that have multiple
clients attached to a bouncer. In this scenario, when users send
messages from one client, the messages get automatically relayed to
other attached clients. This allows all attached clients to display
full conversation.

Servers MAY send a faked message back to the client instead, in cases
where the final version is silently blocked, for example when filtering
spam.

## Limitations

To prevent perceived input lag, clients may choose not to wait for an acknowledgment from the server before displaying sent messages to users. Clients may instead choose to display a temporary local message that is replaced once the `echo-message` acknowledgment is received. However, correlating these is not straightforward, as the server may modify them before acknowledgment. There are also additional complications for self-targeted private messages.

The [`labeled-response`](./labeled-response.html) specification addresses this limitation.

## Examples

In the following examples, `example!ex@example.com` presents a client
that has enabled the `echo-message` capability.

    --> PRIVMSG Attila :hi
    :example!ex@example.com PRIVMSG Attila :hi

The client interprets the received message as if it had sent a `PRIVMSG`
with contents "hi" to `Attila`.

Another example where a server modifies a message by filtering out text
formatting and sends the final version back:

    --> PRIVMSG #ircv3 :back from \02lunch\0F
    :example!ex@example.com PRIVMSG #ircv3 :back from lunch

## Errata

Previous versions of this specification didn't include the Limitations section.

Previous versions of this specification didn't allow for faked messages when filtering spam.
