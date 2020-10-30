---
title: IRCv3 WebSocket support
layout: spec
work-in-progress: true
copyrights:
  -
    name: Darren Whitlen
    period: 2018
    email: darren@kiwiirc.com
---

## Introduction

This specification describes a mechanism for WebSocket-based clients (in particular, web-based IRC clients running in the browser) to connect to IRC networks, with a focus on simplicity and contemporary best practices for web development.

## WebSocket features and encoding

WebSocket is a message-based protocol offering both text (UTF8-encoded) and binary (8-bit clean) message types. Given that the primary purpose of IRC is text-based communication, that web browsers are pushing increasingly towards UTF8 as the default encoding for HTML5 documents and scripting languages, and that these web browsers are the primary use case for IRC-over-WebSockets, we require that all IRC messages MUST be sent as WebSocket text messages, where each message consists of a single complete IRC message line.

The WebSocket protocol handles message fragmentation and termination internally. This is outside the scope of the present specification.

If an IRC server supports both WebSocket and non-WebSocket clients, it may receive non-UTF8 message content from a non-WebSocket client. Servers MUST NOT relay non-UTF8 content to WebSocket clients, since this may result in those clients being disconnected by the browser. Servers MAY replace non-UTF8 bytes with the UTF8 encoding of the Unicode replacement character `�` (`U+FFFD`), or they MAY discard such messages entirely and report an error to the client.

## Message syntax

The syntax of IRC lines MUST NOT be changed, except that servers and clients MUST NOT include trailing `\r` or `\n` characters in their WebSocket messages.

## Client example
~~~
let socket = new WebSocket('wss://ws.example.org');
socket.addEventListener('open', () => {
	socket.send('USER 0 0 0 0');
	socket.send('NICK bob');
});
socket.addEventListener('message', event => processLine(event.data));
~~~

## Security considerations

Clients and servers SHOULD impose limits on the maximum size of messages they will accept, in order to prevent denial-of-service attacks.
