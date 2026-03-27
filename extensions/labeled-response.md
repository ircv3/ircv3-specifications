---
title: Labeled Responses
layout: spec
copyrights:
  -
    name: "Alexey Sokolov"
    period: "2015"
    email: "alexey-irc@asokolov.org"
  -
    name: "James Wheare"
    period: "2016-2020"
    email: "james@irccloud.com"
---

## Introduction

This specification adds a new message tag sent by clients and repeated by servers to correlate responses with a specific request.

## Motivation

Certain client actions can result in responses from the server that vary in interpretation depending on how they were triggered, or otherwise lack a robust way to correlate with local state. Clients have historically needed to keep track of additional local state and/or apply comparison heuristics to server responses to correlate these appropriately.

Labeled responses enable a much simpler form of correlation by using a single id attached to a client request and repeated by the server in its response.

Additionally, labeled responses allow bouncers with multiple connected clients to direct responses (such as WHOIS queries or error messages, see examples) to the correct recipient.

## Architecture

### Dependencies

This specification depends on the [`batch`](../extensions/batch.html) capability which MUST be negotiated to use labeled responses. The order of capability negotiation is not significant and MUST not be enforced.

This specification also uses the [message tags](../extensions/message-tags.html) framework.

### Capabilities

This specification adds the `labeled-response` capability.

Clients requesting this capability indicate that they are capable of handling the message tag, batch type, and `ACK` response described below from servers.

Servers advertising this capability indicate that they are capable of handling the message tag described below from clients, and will use the same tag and value in their response. They will also send the batch type and `ACK` response described below where required.

The value, if present, MUST be a comma (`,`) separated list of flags. The `all` flag is a guarantee that the server WILL return a labeled response for every single command (i.e. the _"where it's feasible to do so"_ language below doesn't apply – there are NO cases where the client can send a command and receive no response, even for asynchronous remote commands).

### Batch types

This specification adds the `labeled-response` batch type, described below.

### Tags

This specification adds the `label` message tag, which has a required value.

This tag MAY be sent by a client for any messages that need to be correlated with a response from the server.

For any message received from a client that includes this tag, the server MUST include the same tag and value in any response required from this message where it is feasible to do so. Servers MUST include the tag in exactly one logical message.

If a response consists of more than one message, a batch MUST be used to group them into a single logical response. The start of the batch MUST be tagged with the `label` tag. The batch type MUST be one of:

* An existing type applicable to the entire response
* `labeled-response`

When a client sends a message to itself, the server MUST NOT include the label tag, except for any acknowledgment sent with the [`echo-message`](/specs/extensions/echo-message.html) mechanism. This allows clients to differentiate between the echoed message response, and the delivered message.

#### Tag value

The tag value is chosen by the client and MUST be treated as an opaque identifier. The client SHOULD NOT reuse a tag value until it has received a complete response for that value from the server. The value MUST NOT exceed 64 bytes.

### The `ACK` response

Servers MUST respond with a labeled `ACK` message when a client sends a labeled command that normally produces no response. It takes no additional parameters and is defined as follows

    :irc.example.com ACK

## Client implementation considerations

This section is non-normative.

There are some cases where a server might not produce a labeled response, or even an `ACK`. Consider the example of an asynchronous command such as a remote `WHOIS nick nick` query that's forwarded to one or more other servers. If the command fails, for instance due to a netsplit, there are several potential outcomes:

* the local server notices that the remote server is unavailable and sends a labeled `ACK` instead of a `WHOIS` response.
* the local server is unable to tell whether the remote server will respond or not and sends neither an `ACK` nor a `WHOIS` response.
* the local server has already begun responding when the netsplit occurs, so the batched response ends early.
* the local server receives a response from the remote server after the netsplit resolves, and sends a `WHOIS` response to the client, potentially without any label.

Clients should handle these cases as they would normally for a server without support for labeled responses.

In the case of `echo-message` (see example below), a client can use labeled responses to correlate a server's acknowledgment of their own messages with a temporary message displayed locally. The temporary message can be displayed to the user immediately in a pending state to reduce perceived lag, and then removed once a labeled response from the server is received.

When a client sends a private message to its own nick, `echo-message` will result in duplicate messages being sent by the server, as both sent and received messages. Labeled responses allow clients to deduplicate these messages in one of two ways:

1. Ignore the labeled message, and use any unlabeled message as acknowledgment for all sent messages to clear temporary local messages.
2. Ignore the unlabeled message.

Both methods assume that the server will acknowledge all successful messages, or return a labeled error response, but differ in their attitude towards the semantics of sending and receiving.

## Bouncer implementation considerations

This section is non-normative.

Bouncers might choose to restrict routing labeled responses to some or all of their clients. For example, the response to a labeled WHOIS request might only be routed back to the originating client. However, an `echo-message` response might be routed to all clients. The bouncer might also choose to make labeled requests to the server on its own behalf, without routing any response to clients.

To avoid label value clashes from multiple connected clients, a bouncer could choose to modify the client supplied label, for instance, by adding a client id prefix. In this case, the modified label would be converted back to its original form before routing the response back to the client.

## Examples

This section is non-normative.

`echo-message` showing a message that has been modified by the server to remove formatting

    Client: @label=pQraCjj82e PRIVMSG #channel :\x02Hello!\x02
    Server: @label=pQraCjj82e :nick!user@host PRIVMSG #channel :Hello!

---

Failed `PRIVMSG` with `ERR_NOSUCHNICK`

    Client: @label=dc11f13f11 PRIVMSG nick :Hello
    Server: @label=dc11f13f11 :irc.example.com 401 * nick :No such nick/channel

---
    
`WHOIS` response using a batch

    Client: @label=mGhe5V7RTV WHOIS nick
    Server: @label=mGhe5V7RTV :irc.example.com BATCH +NMzYSq45x labeled-response
    Server: @batch=NMzYSq45x :irc.example.com 311 client nick ~ident host * :Name
    ...
    Server: @batch=NMzYSq45x :irc.example.com 318 client nick :End of /WHOIS list.
    Server: :irc.example.com BATCH -NMzYSq45x

A server replying with `ACK` where no response is required

    Server: PING :foobar
    Client: @label=abc PONG :foobar
    Server: @label=abc :irc.example.com ACK

## Alternatives

For the use case of bouncers directing query responses to the appropriate client, there exists prior art in the form of the znc module [route_replies](http://wiki.znc.in/Route_replies). The complexities and limitations of that module were a primary motivation for the standardised approach described in this specification.
