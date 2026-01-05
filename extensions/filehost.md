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

This specification introduces the `draft/FILEHOST` isupport token.

Its value MUST be a URI and SHOULD use the `https` scheme. Clients MUST ignore
tokens with an URI scheme they don't support. Clients MUST refuse to use
unencrypted URI transports (such as plain `http`) if the IRC connection is
encrypted (e.g. via TLS).

Servers MUST accept OPTIONS requests on the upload URI. Servers MAY return an
`Accept-Post` header field to indicate the MIME types they accept.

When clients wish to upload a file using the server's recommended service, they
can send a request to the upload URI. The request method MUST be POST. Clients
SHOULD authenticate their HTTP request with the same credentials used on the
IRC connection (e.g. HTTP Basic for SASL PLAIN, HTTP Bearer for SASL
OAUTHBEARER). Clients SHOULD use the `Content-Type`, `Content-Disposition` and
`Content-Length` header fields to indicate the MIME type, name and size of the
file to be uploaded.

On success, servers MUST reply with a `201 Created` status code and with a
`Location` header field indicating the URI of the uploaded file. Servers MUST
support HEAD and GET requests on the uploaded file URI.

Clients SHOULD gracefully handle other common HTTP status codes that could
occur.

## Examples

Example isupport token:

    :irc.example.org 005 seunghye draft/FILEHOST=https://irc.example.org/upload

Example OPTIONS response:

    HTTP/1.1 204 No Content
    Allow: OPTIONS, POST
    Accept-Post: image/*, video/*

Example POST request:

    POST /upload HTTP/1.1
    Host: irc.example.org
    Content-Type: image/jpeg
    Content-Disposition: attachment; filename="picture.jpeg"
    Content-Length: 4242
    Authorization: Basic c2V1bmdoeWU6bm8=

Example POST response:

    HTTP/1.1 201 Created
    Location: /upload/hoh5eFThae4e.jpeg
