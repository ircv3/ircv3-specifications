---
title: `+icon` client tag
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
unprefixed `+icon` tag name. Instead, implementations SHOULD use the
`+draft/icon` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification defines a client-only message tag to indicate message icons.

## Motivation

To allow for richer display of messages with added visual context, a method for client and bot users to attach custom icons to messages is needed. This method is intended for message specific icons. For persistent avatars, a metadata key would be more appropriate.

Visual clients are given a way to choose an appropriate icon resource for the display context, with a method for providing multiple icon sources, with different dimensions.

A content type hint is also provided, to allow clients to filter unsupported image formats without downloading the resource.

## Architecture

### Dependencies

Clients wishing to use this tag MUST negotiate the [`draft/message-tags`](../core/message-tags-3.3.html) capability with the server.

### Format

The icon tag is sent by a client with the client-only prefix `+` and its value is a JSON encoded array of image objects that reference icon resources for the message.

    +draft/icon=<json>

Image objects MAY contain any of the following members:

| Member           | Description |
| :--------------- | :---------- |
| `src` (string)   | The Uniform Resource Indicator (URI) of an icon resource |
| `sizes` (string) | A space separate list of pixel image dimensions in `wxh` format supported by the resource, or a value of `any` for scalable icons. Clients MAY use this to select an appropriate icon resource for the context and pixel density of the display |
| `type` (string)  | A media type hint for the icon resource. Clients MAY use this to filter out unsupported icon types without downloading the resource |

When an icon `src` is an HTTP resource, it SHOULD be referenced with an HTTPS protocol where available.

When the icon is encoded as an [RFC2397 Data URI](https://tools.ietf.org/html/rfc2397) the length of the URI SHOULD NOT exceed 1024 characters.

Icon dimensions SHOULD be square.

## Client implementation considerations

This section is non-normative

Client tags are untrusted data and icon URIs might point to any arbitrary resource. Some examples of unexpected icon conditions and potential mitigations:

* Large file sizes: Validate file sizes and abort if appropriate limits are reached.
* Unexpected dimensions: Scale or crop images to fit within allocated user interface space.
* Unknown formats: Validate content types and reject unaccepted image formats.
* Malicious content: Allow users to block specific icon URIs or icons from specific users.
* Revealing IP addresses to 3rd parties: Add an option to only download icons from trusted sources.

## Examples

This section is non-normative. Indented line breaks are added for readability.

An icon tag describing a single resource with unknown size or type

    @+draft/icon=[{"src":"https://example.com/icon.png"}] PRIVMSG #channel :Example text

---

An icon tag with multiple PNG icon resources of different sizes

    @+draft/icon=[
        {"src":"https://example.com/icon16.png","sizes":"16x16","type":"image/png"},
        {"src":"https://example.com/icon32.png","sizes":"32x32","type":"image/png"},
        {"src":"https://example.com/icon64.png","sizes":"64x64","type":"image/png"}
        ] PRIVMSG #channel :Example text

---

An icon tag with alternate GIF and PNG icon resources

    @+draft/icon=[
        {"src":"https://example.com/icon16.gif","sizes":"16x16","type":"image/gif"},
        {"src":"https://example.com/icon16.png","sizes":"16x16","type":"image/png"}
        ] PRIVMSG #channel :Example text

---

An icon tag with an SVG icon resource usable at any size

    @+draft/icon=[
        {"src":"https://example.com/icon.svg","sizes":"any","type":"image/svg+xml"}
        ] PRIVMSG #channel :Example text

---

A build notifier bot using separate icons for failures and successes

    @+draft/icon=[{"src":"https://example.com/failure.png"}] PRIVMSG #channel :Build 140 failed
    @+draft/icon=[{"src":"https://example.com/success.png"}] PRIVMSG #channel :Build 141 succeeded

---

In this example, a bot client responds to a link sent to the channel by the server by fetching the title and favicon of the link. It then sends a message back to the channel with the link title in the message body and favicon included as a message tag:

    Server: :nick!user@example.com PRIVMSG #channel :https://example.com/a-news-story
    Client: @+draft/icon=[
        {"src":"https://example.com/favicon16.png","sizes":"16x16","type":"image/png"}
        ] PRIVMSG #channel :Example.com: A News Story
