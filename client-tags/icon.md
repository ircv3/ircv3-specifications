---
title: `+icons` client tag
layout: spec
work-in-progress: true
copyrights:
  -
    name: "James Wheare"
    period: 2016
    email: "james@irccloud.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `+icons` tag name. Instead, implementations SHOULD use the
`+draft/icons` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification defines a client-only message tag to indicate message icons

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the [`draft/message-tags`](../core/message-tags-3.3.html) capability with the server.

### Format

The icons tag is sent by a client with the client-only prefix `+` and its value is a JSON encoded array of image objects that reference icon resources for the message.

    +draft/icons=<json>

Image objects MAY contain any of the following members:

| Member           | Description |
| :--------------- | :---------- |
| `src` (string)   | The Uniform Resource Indicator (URI) of an icon resource |
| `sizes` (string) | A space separate list of pixel image dimensions in `wxh` format supported by the resource. Clients MAY use this to select an appropriate icon resource for the context and pixel density of the display |
| `type` (string)  | A media type hint for the icon resource. Clients MAY use this to filter out unsupported icon types without downloading the resource |

When an icon `src` is an HTTP resource, it SHOULD be referenced with an HTTPS protocol where available.

When the icon is encoded as an [RFC2397 Data URI](https://tools.ietf.org/html/rfc2397) the length of the URI SHOULD be no more than 1024.

Icon dimensions SHOULD be square.

## Client implementation considerations

This section is non-normative

Client tags are untrusted data and icon URIs might point to any arbitrary resource. Some examples of unexpected icon conditions and potential mitigations:

* Large file sizes: Validate file sizes and abort if appropriate limits are reached.
* Unexpected dimensions: Scale or crop images to fit within allocated user interface space.
* Unknown formats: Validate content types and reject unaccepted image formats.
* Malicious content: Allow users to block specific icon URIs or icons from specific users.

## Examples

In this example, a bot `C` responds to a link sent to the channel by the server `S` by fetching the title and favicon of the link. It then sends a message back to the channel with the link title in the message body and favicon included as a message tag:

```
S :nick!user@example.com PRIVMSG #channel :https://example.com/a-news-story
C @+draft/icons=[{"src":"https://example.com/favicon16.png","sizes":"16x16","type":"image/png"},{"src":"https://example.com/favicon32.png","sizes":"32x32","type":"image/png"}],{"src":"https://example.com/favicon64.png","sizes":"64x64","type":"image/png"}] PRIVMSG #channel :Example.com: A News Story
```

An example of an icons tag describing a single resource with unknown size or type

```
C @+draft/icons=[{"src":"https://example.com/favicon16.png"}] PRIVMSG #channel :Example text
```
