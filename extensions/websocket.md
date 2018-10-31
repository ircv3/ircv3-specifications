---
title: IRCv3 websocket support
layout: spec
work-in-progress: true
copyrights:
  -
    name: Darren Whitlen
    period: 2018
    email: darren@kiwiirc.com
---

## Introduction

Web based IRC clients are becoming more common and they all increase the demand for websocket connectivity to IRC networks.

This specification provides a common implementation for web and other websocket clients to connect to IRC networks using common and standard development tools used on web pages, with a focus on a simple client facing interface.

## Websocket features and encoding

Websockets offer different transports of communication, binary and strings. With the consideration that the IRC protocol is entirely lines of text, we make use of the message based text implementation that websockets natively provide. These are UTF8 encoded text messages where one websocket message consists of a single complete IRC message line.

The internal websocket protocol handles message fragmentation and termination via frames, which is out of scope for this document.

Because web browsers are pushing more towards a UTF8 default encoding for HTML5 documents and scriptings languages, and these web browsers are the primary use case for websockets, IRC servers MUST encode messages as UTF8 before sending them to the client. Ignoring this may result in the browser closing the websocket connection on invalid UTF8 sequences.

## Message syntax

The syntax of IRC lines MUST NOT be changed. However, IRC servers and clients MUST NOT include trailing new lines in their websocket messages as these are obsolete by websockets native messages.

## Client example
~~~
let socket = new WebSocket('wss://ws.example.org');
socket.addEventListener('open', () => {
	socket.send('USER 0 0 0 0');
	socket.send('NICK bob');
});
socket.addEventListener('message', event => processLine(event.data));
~~~
