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

Software implementing this work-in-progress specification MUST NOT use the unprefixed `EMOJI` ISUPPORT name or the `emoji` channel METADATA name. Instead, implementations SHOULD use the `draft/EMOJI` ISUPPORT name and the `draft/emoji` channel METADATA name to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use unprefixed ISUPPORT and channel METADATA names.

## Introduction

This specification introduces the ability for servers and channels to define packs of custom emoji associated with shortcodes. These packs are accessible via URLs returning a JSON document following a well-defined schema.

## Dependencies

If the server advertises the [`metadata-2`](metadata.html) capability, clients MUST negotiate this capability with the server so they can interact with custom emoji packs defined on a per-channel basis.

## Format

A custom emoji is defined by the following ABNF grammar.

```
emoji-value = ":" shortcode ["/" pack-id] ":"
shortcode   = 1*ustr
pack-id     = 1*(ustr / "/")
ustr        = %x0021-002E   / %x0030-0039   / %x003B-007E   / %x00A1-00AC   / %x00AE-05FF
            / %x0606-061B   / %x061D-06DC   / %x06DE-070E   / %x0710-088F   / %x0892-08E1
            / %x08E3-167F   / %x1681-180D   / %x180F-1FFF   / %x2010-2027   / %x2030-205E
            / %x2065        / %x2070-2FFF   / %x3001-D7FF   / %x7000-FEFE   / %xFFF0-FFF8
            / %xFFFC-110BC  / %x110BE-110CC / %x110CE-1342F / %x13440-1BC9F / %x1BCA4-1D172
            / %x1D17B-E0000 / %xE0002-E001F / %xE0080-10FFFF
            ; All Unicode characters **EXCEPT FOR** Cc (Control), Cf (Format), Cs (Surrogate),
            ; Zl (Line Separator), Zp (Paragraph Separator), Zs (Space Separator),
            ; as well as "/" (solidus, U+002F) and ":" (colon, U+003A)
            ; This corresponds to the regex class [^\p{Cc}\p{Cf}\p{Cs}\p{Zl}\p{Zp}\p{Zs}/:]
```

When receiving a PRIVMSG, NOTICE, or [reaction](../client-tags/react.html), implementations search for all occurrences of emoji-value, and MAY replace those strings with the associated custom emoji image for the provided shortcode from the emoji pack whose id matches pack-id. Implementations SHOULD perform Unicode normalization using Form C (canonical decomposition followed by canonical composition) on the message before attempting emoji replacements. If a given emoji-value did not specify a particular pack-id, implementations MUST use the following algorithm to determine the corresponding pack-id:

1. If the message is in a channel, look up the shortcode within the emoji pack document specified by that channel's `draft/emoji` METADATA.
2. If step 1 fails to produce a shortcode mapping, look up the shortcode within the emoji pack document specified by ISUPPORT `draft/EMOJI`.
3. If steps 1 and 2 fail to produce a shortcode mapping, an implementation MAY attempt to resolve the pack-id via other means (such as a local cache of known packs).
4. If a shortcode is defined in multiple emoji packs within the same emoji pack document, implementations MUST use the pack-id corresponding to the final occurrence of that shortcode.

If an emoji-value specifies a pack-id beginning with a channel prefix, implementations SHOULD query the `draft/emoji` channel METADATA for the channel whose name matches pack-id and use the emoji pack document from that metadata value, if any, to resolve the shortcode to a custom emoji image.

Whether any particular emoji-value is replaced with an associated custom emoji is an implementation decision (see Client implementation considerations below).

*[[Begin non-normative example--*

For example, a client receives the following message:

    :nick!user@example.com PRIVMSG #channel ::wave: Hello! :smile/#otherchannel:

The client will attempt to replace `:wave:` first with an emoji with the shortcode `wave` specified in the emoji pack document linked to by `METADATA #channel GET draft/emoji`. If no emoji packs exist in that metadata, the client then attempts to resolve the `wave` shortcode from the emoji pack document linked to by RPL_ISUPPORT `draft/EMOJI`. If that also fails, the client uses any other internal mappings it wishes (for example, the user may have explicitly installed an emoji pack to the client providing the `wave` shortcode, or a client may have a default mapping of `wave` to the standard 👋 emoji).

The client then attempts to replace `:smile/#otherchannel:` with an emoji whose pack-id equals `#otherchannel`. Since this pack-id begins with a channel type prefix, the client first attempts to resolve the pack by obtaining the emoji pack document linked to by `METADATA #otherchannel GET draft/emoji`, and if that document specifies the `#otherchannel` emoji pack with a `smile` shortcode, the client uses it.

*--End non-normative example]]*

## Emoji pack document

An emoji pack document is a valid JSON array containing zero or more emoji packs, as described below. All strings in an emoji pack document MUST be valid UTF-8 and MUST be normalized using Unicode normalization form C (canonical decomposition followed by canonical composition). If any portion of an emoji pack document is invalid, the behavior is implementation-defined. For example, a client MAY choose to disregard the entire document, even if it contains some valid packs, or it may choose to only disregard invalid packs.

An emoji pack is a JSON object containing the following keys:

- `id` (string): An emoji pack MUST contain this key. It is an internal identifier and MUST be unique across all emoji packs in the emoji pack document. The value MUST be a string conforming to the pack-id production of the ABNF grammar in the Format section.
- `name` (string): An emoji pack MUST contain this key. It is a display name for the pack to be used by clients in their UIs wherever they see fit. The value MUST be a string of at least 1 byte long and SHOULD NOT contain newlines or other non-printable characters. Clients MAY truncate or alter the value for proper display in their UIs.
- `description` (string): An emoji pack SHOULD contain this key. If it does, the value MUST be a string of any length. The value for this key is a description of the emoji pack.
- `authors` (array): An emoji pack SHOULD contain this key. If it does, the value MUST be an array with 0 or more items, and all array items MUST be strings. Each array item is an author for the emoji pack and/or the emojis within the pack. This specification does not define any particular format for the strings within the author list, and clients SHOULD expect the values to vary wildly.
- `homepage` (string): An emoji pack MAY contain this key. If it does, the value MUST be a string that is a valid URL a user can visit to learn more about the pack.
- `required` (array): An emoji pack MAY contain this key. If it does, the value MUST be an array with 0 or more items, and all array items MUST be strings. The value describes specification extensions that the client MUST understand in order to properly use this emoji pack. If a client does not understand all of the items, it MUST NOT attempt to use, install, or render this emoji pack.
- `emoji` (object): An emoji pack MUST contain this key. The value MUST be an object mapping emoji shortcodes to the data for that emoji. Object keys are shortcodes, which MUST be strings conforming to the shortcode production of the ABNF grammar in the Format section. The values of shortcode keys MUST an emoji object. The `emoji` object MUST contain at least one valid shortcode to emoji object mapping.

An emoji object is a JSON object containing the following keys:

- `url` (string): An emoji object MUST contain this key. Its value MUST be a string containing a valid URL to obtain the emoji image data.
- `alt` (string): An emoji object SHOULD contain this key. Its value MUST be a string of any length and provides replacement text to use for when the image is unavailable.

To allow for future extensibility, clients MUST ignore unrecognized keys within emoji packs and emoji objects. This specification intentionally does not define any mechanisms to create or manage emoji pack documents. Other specifications may define such mechanisms.

*[[Begin non-normative example--*

An example emoji pack document which defines two emoji packs is below. The second pack contains animated emoji using the hypothetical vendored example.com/animated requirement as well as an additional "license" property not defined by this specification, which clients must ignore if they do not recognize it.

```json
[
    {
        "id": "#coolchannel",
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

If a client is unwilling or unable to render any particular emoji image, they have a choice of displaying the alt text or leaving the shortcode as-is in the message body. In general, prefer to display the alternate text (even if it is empty). This could be in conjunction with a "broken image" placeholder where the alt text is presented via tooltip. However, if the shortcode does not map to any known valid emoji, then the shortcode should be left as-is in the message (as it likely is referring to something other than an emoji, such as a snippet of programming source code). Clients may also choose to leave shortcodes as-is even if they map to valid emoji if they are found within certain contexts such as multiline code blocks.

When processing [multiline](multiline.html) batches, an open question becomes whether emoji replacement is performed before or after combining lines tagged with `+draft/multiline-concat` together. Because shortcodes and pack-ids have no length constraints, it is entirely possible for a client to choose to split an emoji into multiple lines of a multiline batch. As such, it is also probably a good idea to perform emoji replacement on the final multiline message (after concatenating all lines tagged with `+draft/multiline-concat` and then peforming Unicode normalization with NFC on the result of that operation). This could result in emoji being displayed for clients supporting multiline while they would not be displayed for clients without multiline support due to the line splitting, but the resulting user experience is arguably better for clients which do support multiline.

## Server implementation considerations

This section is non-normative.

A server may wish to rewrite URLs to and within emoji pack documents to a server-controlled space, downloading the JSON and all relevant images locally. This allows for vetting of pack ids (if the server wishes to enforce any particular naming convention for pack ids to prevent channels from extending or overriding server packs), virus scans on images, stronger guarantees that the URL will remain available for clients to fetch, and the ability to prevent clients from leaking IP addresses to third party websites.

## Emoji pack author considerations

This section is non-normative.

A pack-id matching the channel name for emoji defined in channel metadata allows those emoji to be "shared" between multiple channels. If this behavior is undesirable, avoid using the channel name as a pack-id. When filling the `alt` property for custom emoji, the [alt attribute decision tree](https://www.w3.org/WAI/tutorials/images/decision-tree/) might be useful to determine how to define the property value.

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
                "propertyNames": {"pattern": "^[^\\p{Cc}\\p{Cf}\\p{Cs}\\p{Zl}\\p{Zp}\\p{Zs}/:]+$"},
                "additionalProperties": {
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
                "minProperties": 1
            }
        }
    }
}
```