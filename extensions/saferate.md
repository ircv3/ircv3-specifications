---
title: SAFERATE ISUPPORT token
layout: spec
work-in-progress: true
copyrights:
  -
    name: "delthas"
    period: "2024"
---

## Introduction

This is a work-in-progress specification.

This specification offers a way for servers to advertise that clients will not
be disconnected due to server rate-limiting / anti-flood, and so that the client
can send messages without doing its own conservative rate-limiting.

## The `SAFERATE` ISUPPORT token

This specification introduces the `SAFERATE` ISUPPORT token.

When advertised, the server ensures that a client will not be disconnected due
to server rate-limiting.

The token MUST NOT be advertised with a value.

## Implementation considerations

In order to support this specification, a server can use fakelag to delay
processing new messages rather than disconnect a client. The queue of incoming
messages to process should be bounded.

A client can disable its internal rate limiting when receiving this token.
