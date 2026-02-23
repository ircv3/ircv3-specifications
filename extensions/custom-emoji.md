---
title: "Custom Emoji"
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Ryan Schmidt"
    period: "2026"
    email: "moonmoon@libera.chat"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `EMOJI` ISUPPORT name, the `emoji` channel METADATA name, or the `+emoji` tag name. Instead, implementations SHOULD use the `draft/EMOJI` ISUPPORT name, the `draft/emoji` channel METADATA name, and the `+draft/emoji` tag name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use unprefixed ISUPPORT, channel METADATA, and tag names.

## Introduction

This specification introduces the ability for servers and channels to define packs of custom emoji associated with shortcodes. These packs are accessible via URLs returning a JSON document following a well-defined schema.

## Dependencies

Clients wishing to display or send custom emoji MUST negotiate the [`message-tags`](message-tags.html) capability with the server. Clients SHOULD additionally negotiate the [`metadata-2`](metadata.html) capability with the server so they can interact with custom emoji packs defined on a per-channel basis.

## Format

The emoji tag is sent by a client with the client-only prefix `+`. The value is a comma-separated (`,`) list of shortcode=packid pairs, with an optional suffix for each shortcode, as defined by the following ABNF grammar.

```
value       = emoji-def *("," emoji-def)
emoji-def   = shortcode ["~" suffix] "=" pack-id
shortcode   = 1*40(ALPHA / DIGIT / %x21-2B / %x2E-2F / %x3B-3C / %x3E-3F / %x5B-60 / %x7B-7D)
              ; All ASCII numbers, letters, and symbols EXCEPT FOR ,=@:~
suffix      = 1*3DIGIT
pack-id     = 1*(ALPHA / DIGIT / %x21-2B / %x2E-2F / %x3A-3C / %x3E-40 / %x5B-60 / %x7B-7E)
              ; All ASCII numbers, letters, and symbols EXCEPT FOR ,=
```

The ordering of pairs within the tag is not meaningful. The same shortcode and suffix combination MUST NOT appear more than once within the emoji tag. Implementations receiving the same shortcode and suffix combination multiple times SHOULD disregard all but the final occurrence.

Implementations MUST treat shortcodes and suffixes as case-sensitive opaque identifiers and MUST NOT perform any validation that would reject the message if an invalid shortcode or suffix is used, although implementations SHOULD disregard such shortcodes when performing emoji replacement described below. This allows future modifications to the format.

When receiving a PRIVMSG, NOTICE, or [reaction](../client-tags/react.html) with an emoji tag, clients search for all occurrences of the exact substrings of each shortcode and optional suffix surrounded by a colon (`:`) on each side outside of monospaced contexts, and MAY replace those strings with the associated custom emoji image for the provided shortcode from the emoji pack whose id matches pack-id. Whether any particular shortcode is replaced with an associated custom emoji is an implementation decision for the client (see Client implementation considerations below).

A monospaced context is a portion of the message body beginning with the monospace formatting code 0x11 and ending with either the monospace formatting code 0x11 again or the format reset code 0x0F. Even if a client does not support 0x11 for formatting text as monospaced, it MUST NOT replace custom emoji shortcodes within such contexts with the associated images. This allows for an easy way of "escaping" emoji shortcodes that would otherwise be replaced by images.

*[[Begin non-normative example--*

For example, upon receiving a message with the tag `+draft/emoji=lol=server,lol~1=#coolpeople/default,lol~01=#coolpeople/pack2`, the client would search for the string `:lol:` within the message body and replace all occurrences of that string outside of monospaced contexts with the `lol` emoji from the `server` emoji pack. It will similarly replace `:lol~1:` with the `lol` emoji from the `#coolpeople/default` emoji pack and `:lol~01:` with the `lol` emoji from the `#coolpeople/pack2` emoji pack.

*--End non-normative example]]*

## Emoji pack document

An emoji pack document is a valid JSON array containing zero or more emoji packs, as described below. All strings in an emoji pack document MUST be valid UTF-8. If any portion of an emoji pack document is invalid, the behavior is implementation-defined. For example, a client MAY choose to disregard the entire document, even if it contains some valid packs, or it may choose to only disregard invalid packs.

An emoji pack is a JSON object containing the following keys:

- `id` (string): An emoji pack MUST contain this key. It is an internal identifier and MUST be unique across all emoji packs in the emoji pack document. The value MUST be a string conforming to the pack-id production of the emoji tag ABNF grammar in the Format section.
- `name` (string): An emoji pack MUST contain this key. It is a display name for the pack to be used by clients in their UIs wherever they see fit. The value MUST be a string of at least 1 byte long and SHOULD NOT contain newlines or other non-printable characters. Clients MAY truncate or alter the value for proper display in their UIs.
- `description` (string): An emoji pack SHOULD contain this key. If it does, the value MUST be a string of any length. The value for this key is a description of the emoji pack.
- `authors` (array): An emoji pack SHOULD contain this key. If it does, the value MUST be an array with 0 or more items, and all array items MUST be strings. Each array item is an author for the emoji pack and/or the emojis within the pack. This specification does not define any particular format for the strings within the author list, and clients SHOULD expect the values to vary wildly.
- `homepage` (string): An emoji pack MAY contain this key. If it does, the value MUST be a string that is a valid URL a user can visit to learn more about the pack.
- `required` (array): An emoji pack MAY contain this key. If it does, the value MUST be an array with 0 or more items, and all array items MUST be strings. The value describes specification extensions that the client MUST understand in order to properly use this emoji pack. If a client does not understand all of the items, it MUST NOT attempt to use, install, or render this emoji pack.
- `emoji` (object): An emoji pack MUST contain this key. The value MUST be an object mapping emoji shortcodes to the data for that emoji. Object keys beginning with the at sign (`@`) are treated as metadata for later extensions to this specification and MUST NOT be interpreted by the client as emoji shortcodes. All other keys are shortcodes, which MUST be strings conforming to the shortcode production of the ABNF grammar in the Format section. The values of shortcode keys MUST an emoji object. The `emoji` object MUST contain at least one valid shortcode to emoji object mapping.

An emoji object is a JSON object containing the following keys:

- `url` (string): An emoji object MUST contain this key. Its value MUST be a string containing a valid URL to obtain the emoji image data.
- `alt` (string): An emoji object SHOULD contain this key. Its value MUST be a string of any length and has the same [use-cases as the alt attribute](https://www.w3.org/WAI/tutorials/images/decision-tree/) on an `<img>` HTML element.

To allow for future extensibility, clients MUST ignore unrecognized keys within the emoji pack object.

*[[Begin non-normative example--*

An example emoji pack document which defines two emoji packs is below. The second pack contains animated emoji using the hypothetical vendored example.com/animated requirement as well as an additional "license" property not defined by this specification, which clients must ignore if they do not recognize it.

```json
[
    {
        "id": "#coolchannel/default",
        "name": "#coolchannel",
        "description": "Default emoji pack for #coolchannel",
        "authors": [
            "Author One",
            "Author Two <author2@example.com>",
            "",
            "author3"
        ],
        "homepage": "https://example.com/custom-emoji",
        "emoji": {
            "kekw": {
                "url": "https://example.com/kekw.png"
            },
            "this": {
                "url": "https://example.com/this.jpg",
                "alt": "the word 'this' with an arrow pointing upwards at the preceding message"
            },
            "@comment": "data URL below truncated in this example for brevity",
            "thonk": {
                "url": "data:image/png;base64,iVBORw0KGgoAAAAN...ErkJggg==",
                "alt": ""
            }
        }
    },
    {
        "id": "#coolchannel/animated",
        "name": "#coolchannel Animated Emoji",
        "description": "Animated emoji pack for #coolchannel",
        "authors": [
            "名無しの権兵衛"
        ],
        "homepage": "https://example.com/custom-emoji",
        "license": "CC-BY-SA-4.0",
        "required": [
            "example.com/animated"
        ],
        "emoji": {
            "kekw": {
                "url": "https://example.com/kekw.apng",
                "alt": "laughing"
            }
        }
    }
]
```

*--End non-normative example]]*

## Emoji images

Emoji images SHOULD be a minimum of 128x128 pixels in size for raster images. This allows clients to size emoji appropriately in their UI without needing to worry about blurry images on high DPI displays. This specification does not dictate any particular file formats for emoji images, however clients SHOULD support static PNG and JPG images and MAY support any number of additional formats. See the client implementation considerations section for discussion on the security implications of supporting certain formats.

## Server advertisement (ISUPPORT)

Servers wishing to provide server-wide emoji packs MAY publish a `draft/EMOJI` ISUPPORT token. If they do, the value MUST be the URL of an emoji pack document. The URL SHOULD use the https scheme.

## Channel advertisement (METADATA)

If a channel contains custom emoji, the `draft/emoji` channel metadata key will be set to the URL of an emoji pack document. The pack ids of this document MAY be duplicative of pack ids advertised by the server via ISUPPORT. In this case, when rendering emoji from a duplicated pack and with a duplicated shortcode in that channel, clients SHOULD use the emoji object defined in the channel metadata instead of the emoji object defined at the server level.

## Client implementation considerations

This section is non-normative.

This specification contains URLs for both emoji pack documents as well as emoji images, which may be user-controlled in the case of channel metadata. Visiting arbitrary URLs can expose the user's IP address to the remote server, so clients may wish to alert users before downloading any emoji packs of this risk. The decision to display any particular emoji is intentionally left to the client so they can implement workflows such as "installing" emoji packs before they are displayed, caching emoji images, etc.

While there is no update notification message built into this specification for when emoji or emoji packs are added/removed, both ISUPPORT as well as METADATA support notifications (via the server sending a new RPL_ISUPPORT or via METADATA SUB). If a client receives such an update notification but the URL value is the same value as before, the client should interpret this as an update within the emoji pack document itself and re-fetch the emoji pack document and any linked-to emoji. Since there is no guarantee that such an update notification mechanism is employed by the server, client implementations will likely need to also resort to polling the documents and checking if they have been updated; a HEAD request with If-Modified-Since to the URL can be used to do this checking in a lightweight manner as the remote HTTP server should respond with 304 Not Modified if no modifications have been performed, avoiding expensive re-parsing of a potentially large JSON document.

Clients should implement defensive coding practices around image parsing, as there is a chance image data is malicious. This is especially relevant for SVG support as SVG natively supports scripting capabilities that could run client-side and could contain external XML entities causing additional web traffic or broken parsing, but security vulnerabilities do exist with image processing libraries in general. Clients should be strict in what they accept and reject any malformed image data.

If a client is unwilling or unable to render any particular emoji image, they have a choice of displaying the alt text or leaving the shortcode as-is in the message body. In general, leaving the shortcode as-is is likely the best choice, as it doesn't potentially introduce arbitrarily-long content that users on other clients are unlikely to see. The alt text could still be exposed as a tooltip when hovering over the shortcode, for example.

## Server implementation considerations

This section is non-normative.

A server may wish to rewrite URLs to and within emoji pack documents to a server-controlled space, downloading the JSON and all relevant images locally. This allows for vetting of pack ids (if the server wishes to enforce any particular naming convention for pack ids to prevent channels from extending or overriding server packs), virus scans on images, stronger guarantees that the URL will remain available for clients to fetch, and the ability to prevent clients from leaking IP addresses to third party websites.

## Emoji pack document schema

A [JSON schema](https://json-schema.org/specification) usable for an emoji pack document is below. Note that the schema does not fully capture every requirement described in the emoji pack document section, but is usable as a starting point for validating documents.

```json
{
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "$id": "https://ircv3.net/extensions/custom-emoji.json",
    "title": "Emoji Pack Document",
    "description": "A collection of custom emoji packs for use in IRC clients.",
    "type": "array",
    "minItems": 0,
    "items": {
        "type": "object",
        "required": ["id", "name", "emoji"],
        "properties": {
            "id": {
                "description": "A unique ID for this emoji pack",
                "type": "string",
                "pattern": "^[a-zA-Z0-9!-+\\-./:;<>?@[-`{-~]+$"
            },
            "name": {
                "description": "The display name of the emoji pack",
                "type": "string",
                "minLength": 1,
            },
            "description": {
                "description": "Text describing the emoji pack",
                "type": "string"
            },
            "authors": {
                "description": "Authors of the emoji pack and/or emoji contained within the emoji pack",
                "type": "array",
                "items": {"type": "string"}
            },
            "homepage": {
                "description": "A URL that a user can visit to learn more about the emoji pack",
                "type": "string",
                "format": "uri"
            },
            "required": {
                "description": "Extensions the client MUST understand before using this pack",
                "type": "array",
                "items": {"type": "string", "minLength": 1}
            },
            "emoji": {
                "description": "A mapping of shortcode to emoji URL for emoji in this pack",
                "type": "object",
                "patternProperties": {
                    "^[a-zA-Z0-9!-+\\-./;<>?[-`{|}]{1,40}$": {
                        "type": "object",
                        "required": ["url"],
                        "properties": {
                            "url": {
                                "description": "The URL of the emoji image",
                                "type": "string",
                                "format": "uri"
                            },
                            "alt": {
                                "description": "Alt text of the emoji image for accessibility",
                                "type": "string"
                            }
                        }
                    },
                    "^@.+$": {}
                },
                "additionalProperties": false,
                "minProperties": 1
            }
        }
    }
}
```