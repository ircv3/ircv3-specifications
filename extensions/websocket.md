---
title: IRCv3 WebSocket support
layout: spec
work-in-progress: true
copyrights:
  -
    name: Shivaram Lingamneni
    period: 2021
    email: slingamn@cs.stanford.edu

---

## Introduction

This specification describes a mechanism for WebSocket-based clients (in particular, web-based IRC clients running in the browser) to connect to IRC networks.

## WebSocket features and encoding

WebSocket is a message-based protocol, i.e., it handles message fragmentation and termination internally. It offers both text (UTF-8 encoded) and binary (8-bit clean) message types; each has advantages and disadvantages as a transport for IRC. Binary messages can transport any valid IRC line, while text messages can only transport lines that are entirely UTF-8. Consequently, binary frames are necessary to fully support IRC communities that use encodings other than UTF-8. On the other hand, given that web browsers are pushing increasingly towards UTF-8 as the default encoding for HTML5 documents and scripting languages, text messages make it simpler to achieve a correct implementation on the client side.

We define two [WebSocket subprotocols](https://tools.ietf.org/html/rfc6455#section-1.9): `binary.ircv3.net` and `text.ircv3.net`. Servers MUST support `binary.ircv3.net` and SHOULD support `text.ircv3.net`. Clients connecting to a server providing IRC-over-WebSockets MUST initially request one or both of these subprotocols, in order of preference (most preferred first). The server MUST accept any subprotocol it supports, respecting the client's order of preference. If no subprotocol can be agreed on, the server MAY continue connection establishment without sending the `Sec-WebSocket-Protocol` header.

If `binary.ircv3.net` is negotiated, then client and server will exchange binary messages; if `text.ircv3.net` is negotiated, then they will exchange text messages.  If no subprotocol is successfully negotiated, server and client MAY implement fallbacks designed for compatibility with legacy software; the exact behavior is left implementation-defined. In all cases, the message format is the same: each message MUST consist of a single IRC line, except that servers and clients MUST NOT include trailing `\r` or `\n` characters in the message.

Servers MUST NOT relay non-UTF-8 content to clients using text messages, since this may result in those clients being disconnected by the browser. Servers MAY replace non-UTF-8 bytes with the UTF-8 encoding of the Unicode replacement character `�` (`U+FFFD`).

## Client example
~~~
let socket = new WebSocket('wss://ws.example.org', ['text.ircv3.net']);
socket.addEventListener('open', () => {
	socket.send('USER 0 0 0 0');
	socket.send('NICK bob');
});
socket.addEventListener('message', event => processLine(event.data));
~~~

## Security considerations

Clients and servers SHOULD impose limits on the maximum size of messages they will accept, in order to prevent denial-of-service attacks. The limits SHOULD reflect the increased total line lengths allowed by the [`message-tags`](./message-tags) specification.

## Other implementation considerations

This section is non-normative.

Given the numerous implementations in the wild predating this specification, servers and clients should strive for maximum compatibility.

Servers may wish to support legacy clients that do not negotiate a subprotocol. For example, when no subprotocol is negotiated, servers could accept either incoming message type, while using a fixed (or operator-configurable) message type for outgoing messages. Alternatively, they could autodetect the outgoing message type based on the type of the first incoming message sent by the client.

Similarly, clients may wish to support legacy servers that are unaware of subprotocols. The [standard JavaScript API](https://fetch.spec.whatwg.org/#websocket-opening-handshake) for WebSockets will [fail the WebSocket connection](https://tools.ietf.org/html/rfc6455#section-7.1.7) if none of the client's desired subprotocols could be negotiated. In this case, the client could attempt another connection without subprotocol negotiation.
