---
title: IRCv3.2 Message Tags
layout: spec
copyrights:
  -
    name: "Alexey Sokolov"
    period: "2012-2014"
    email: "alexey-irc@asokolov.org"
  -
    name: "Stéphan Kochen"
    period: "2012"
    email: "stephan@kochen.nl"
  -
    name: "Kyle Fuller"
    period: "2012"
    email: "inbox@kylefuller.co.uk"
---
Additional optional tags are added to the start of each message.

The message pseudo-BNF, as defined in [RFC 1459, section 2.3.1][rfc1459] is extended to look as follows:

	<message>       ::= ['@' <tags> <SPACE>] [':' <prefix> <SPACE> ] <command> <params> <crlf>
	<tags>          ::= <tag> [';' <tag>]*
	<tag>           ::= <key> ['=' <escaped value>]
	<key>           ::= [ <vendor> '/' ] <sequence of letters, digits, hyphens (`-`)>
	<escaped value> ::= <sequence of any characters except NUL, CR, LF, semicolon (`;`) and SPACE>
	<vendor>        ::= <host>

Every message tag is enabled by a capability. The same capability can be used for several tags, if these tags
are intended to be used together.

Every tag can have own rules when it can be used: from client to server only, from server to client only, or in both directions.

Server MUST NOT add a tag to messages, if client didn't request the capability which enables the tag using `CAP REQ`.
Server MUST NOT add a tag to messages before replying to client's `CAP REQ` with `CAP ACK`.
If client requested a capability, which enables one or more message tags, client MUST be able to parse the tags syntax.

Client MUST NOT add a tag to messages before server replies to client's `CAP REQ` with `CAP ACK`.
If server accepted the capability request with `CAP ACK`, server MUST be able to parse the tags syntax.

Client and server MAY parse the tags even without any capabilities enabled.
They SHOULD ignore tags of capabilities which are not enabled.

This ensures back-compatibility with [standard IRC protocol][rfc1459],
because no capabilities are enabled by default,
and therefore messages can't have any tags unless client explicitly enables such capabilities.

Size limit for the message tags MUST be 512 bytes, including `@` and the trailing space characters, leaving 510 bytes for tags themselves.
So the total size of a message becomes limited by 1024 bytes: 512 bytes for message tags, and 512 bytes for the rest as specified by [RFC 1459][rfc1459].

If a message has multiple tags, order of the tags MUST NOT matter.
Two or more tags with the same key MUST NOT appear on the same message.

## Escaping values

The mapping between characters in tag values and their representation in `<escaped value>` MUST be this:

| Character       | Sequence in `<escaped value>` |
|-----------------|-------------------------------|
| `;` (semicolon) | `\:` (backslash and colon)    |
| `SPACE`         | `\s`                          |
| `\`             | `\\`                          |
| `CR`            | `\r`                          |
| `LF`            | `\n`                          |
| all others      | the character itself          |

Reason: more common URL-escaping eats more space, while IRC message's length is very limited.
Also having semicolon as `\:` makes it easy to split the `<tags>` string by `;` first.

## Rules for naming message tags

There are two tag namespaces:

### Vendor-Specific

Names which contain a slash character (`/`) designate a vendor-specific tag namespace.
These names are prefixed by a valid DNS domain name.

For example: `znc.in/server-time`.

In cases if the domain name contains non-ASCII characters, punycode MUST be used,
e.g. `xn--e1afmkfd.org/foo`.

Vendor-Specific tags should be submitted to the IRCv3 working group for consideration.

### Standardized

Names for which a corresponding document sits in the IRCv3 Extension Registry.

Names in the IRCv3 Extension Registry are reserved for your tag.

## Examples

Message with 0 tags:

	:nick!ident@host.com PRIVMSG me :Hello

Message with 3 tags:

	@aaa=bbb;ccc;example.com/ddd=eee :nick!ident@host.com PRIVMSG me :Hello

* standardized tag `aaa` with value `bbb`

* message is tagged with standardized tag `ccc`

* tag `ddd` specific to software of `example.com` with value `eee`



[rfc1459]: http://tools.ietf.org/html/rfc1459#section-2.3.1
