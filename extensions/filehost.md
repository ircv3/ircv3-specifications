---
title: The Filehost ISUPPORT token
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Val Lorentz"
    email: "progval+ircv3@progval.net"
    period: "2022"
  -
    name: "Simon Ser"
    period: "2024"
---

# filehost

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `FILEHOST` ISUPPORT name. Instead, implementations SHOULD use the
`draft/FILEHOST` ISUPPORT name to be interoperable with other software
implementing a compatible work-in-progress version. The final version of the
specification will use unprefixed ISUPPORT names.

## Motivation

This specification offers a way for servers to advertise a hosting service for
users to upload files (such as text or images), so they can post them on IRC.

## Architecture

### ISUPPORT token

This specification introduces the `draft/FILEHOST` ISUPPORT token.

Its value MUST be a URI and SHOULD use the `https` scheme. Clients MUST ignore
tokens with an URI scheme they don't support. Clients MUST refuse to use
unencrypted URI transports (such as plain `http`) if the IRC connection is
encrypted (e.g. via TLS).

Servers MUST accept `OPTIONS` requests on the upload URI. Servers MAY return an
`Accept-Post` header field to indicate the MIME types they accept. Servers MAY
advertise support for [resumable uploads].

When clients wish to upload a file using the server's recommended service, they
can send a request to the upload URI. The request method MUST be `POST`.
Clients MUST include an `Authorization` header field with the `Bearer` scheme
and a fresh token obtained via the `FILEHOST TOKEN` command.

Clients SHOULD use the `Content-Type`, `Content-Disposition` and
`Content-Length` header fields to indicate the MIME type, name and size of the
file to be uploaded.

On success, servers MUST reply with a `201 Created` status code and with a
`Location` header field indicating the URI of the uploaded file. Servers MUST
support `HEAD` and `GET` requests on the uploaded file URI.

Clients SHOULD gracefully handle other common HTTP status codes that could
occur.

### `FILEHOST TOKEN` command

When the `draft/FILEHOST` ISUPPORT token is advertised, servers MUST support
the `FILEHOST TOKEN` command. When sent from the client, the command takes no
parameter:

    FILEHOST TOKEN

The server MUST reply with a successful `FILEHOST TOKEN` or a `FAIL` reply.
A successful `FILEHOST TOKEN` server reply has the following syntax:

    FILEHOST TOKEN <token>

Servers MUST ensure that tokens are short-lived and MAY invalidate tokens once
they are used.

## Examples

Example isupport token:

    :irc.example.org 005 seunghye draft/FILEHOST=https://irc.example.org/upload

Example `OPTIONS` response:

    HTTP/1.1 204 No Content
    Allow: OPTIONS, POST
    Accept-Post: image/*, video/*

Example token query:

    C: FILEHOST TOKEN
    S: FILEHOST TOKEN oophay9HohKeAhb1nohy1iec

Example `POST` request:

    POST /upload HTTP/1.1
    Host: irc.example.org
    Content-Type: image/jpeg
    Content-Disposition: inline; filename="picture.jpeg"
    Content-Length: 4242
    Authorization: Bearer oophay9HohKeAhb1nohy1iec

Example `POST` response:

    HTTP/1.1 201 Created
    Location: /upload/hoh5eFThae4e.jpeg

[resumable uploads]: https://datatracker.ietf.org/doc/draft-ietf-httpbis-resumable-upload/
