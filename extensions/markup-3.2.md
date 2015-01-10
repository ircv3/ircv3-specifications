# IRC Message Markup

Copyright (c) 2015 Edward Powell <me@embolalia.com>.

Unlimited redistribution and modification of this document is allowed
provided that the above copyright notice and this permission notice
remains in tact.

## What message markup attempts to solve

Currently, styled text in IRC messages is achieved through the use of control
codes. These codes are bytes within the range of unprintable ASCII characters,
with color formatting followed by printable characters. Clients must know what
these codes are and handle them, even if they don't support the functionality,
in order to avoid showing garbage to the user. Few, if any, keyboards natively
support the entry of these codes, essentially requiring key bindings for entry.
Moreover, there is no formal specification of how these codes should work,
instead leaving the definition of this behavior to what is supported by clients
*de facto*.

## Overview of markup elements

Message markup is defined through the use of a number of common characters. For
the purposes of the below, text is considered "wrapped" by a string if it is
immediately preceded and immediately followed by the given string, without
containing the string.

* Text which is wrapped by one or more asterisks (`*`) or underscores
  (`_`) SHOULD be rendered as emphasized. If a client is unable to support one
  form of emphasis, another form MAY be supported.
  * Where the text is wrapped by a single asterisk (`*`), it should be rendered as
    italic.
  * Where the text is wrapped by two asterisks (`**`), it should be rendered as
    bold.
  * Where the text is wrapped by a single underscore (`_`), it should be
    rendered as underlined.
* Text wrapped by square brackets (`[` and `]`) which is immediately followed
  by text which is wrapped by parentheses (`(` and `)`) SHOULD be rendered as a
  link.
  * The text of the link MUST be the text within the square brackets.
  * The target of the link MUST be the text within the parentheses.
  * If the target of the link contains a colon (`:`) or period (`.`), it should
    be treated as a URI.
  * If the target of the link contains neither a colon nor a period, it should
    be treated as either a nickname or a channel on the network from which the
    message was sent.
  * If the target of the link is being treated as a URI, any characters which
    are not valid in a URI SHOULD be escaped appropriately. Such parsing MUST
    NOT escape URIs which are already valid.
  * If the target of the link contains a period, but not a colon, it MUST be
    prepended with `http://`.
  * Clients MAY limit rendering URI links to those with certain URI schemes. If
    this is done, the full text MUST be displayed instead, including the URI.
  * If the link appears within a block which is otherwise formatted, the text
    of the link SHOULD be simmilarly formatted. The text of a link MUST NOT
    contain another link.
* Text wrapped by two "at" symbols (`@@`) SHOULD be rendered in a different
  color.
  * This color SHOULD be consistent with colors used for other important text,
    for example messages containing the user's nickname.
  * If the initial pair of symbols is immediately followed by text wrapped in
    parentheses, the text in the parentheses should be parsed as a color.
* Text wrapped by a single backtick (`\``) SHOULD NOT be formatted by any of
  the above rules. It SHOULD be rendered in a monospaced font or, where
  monospaced font is already the default, it SHOULD be differentiated from the
  rest of the text (for example, by a slightly different background color).
  * Text wrapped by three backticks (`\`\`\``) should also be rendered as such,
    noting that single backticks within this text will be rendered literally
    along with all other formatting characters.
* Where any formatting mark is immediately preceeded by a backslash (`\\`)
  which is not, itself, immediately preceeded by a backslash, the character
  (but not the backslash) should be rendered, and the formatting rule should
  not be considered to have been matched.


## Colors

Where the color is specified as one of the numbers or names below, it SHOULD be
rendered in a color simmilar to the one with the RGB color code specified.
Where a two-digit number starting with 0 is used, it MUST be treated as the
appropriate one digit number. A color given as a number MUST be treated the
same as the corresponding name. Color names MUST be case-insensitive.

Clients SHOULD prefer using color names to numbers or codes, especially in user
interfaces. Clients MAY allow users to enter locale-specifc names for colors,
provided that such is translated to the name as given below when sent to the
server.

Except for where bolded, these are identical to the numbers used in practice
(as originally laid out in the mIRC client) and to the names given in the HTML
4.01 specification.

Number | Name       | Hexadecimal code
------ | ---------- | ----------------
0      | White      | #FFFFFF
1      | Black      | #000000
2      | Navy       | #000080
3      | Green      | #008000
4      | Red        | #FF0000
5      | Maroon     | #800000
6      | Purple     | #800080
7      | **Orange** | #FF6600
8      | Yellow     | #FFFF00
9      | Lime       | #00FF00
10     | Teal       | #008080
11     | Aqua       | #00FFFF
12     | Blue       | #0000FF
13     | Fuchsia    | #FF00FF
14     | Gray       | #808080
15     | Silver     | #C0C0C0
**16** | Olive      | #808000

## Pseudo-BNF

    <italicized>  =:: '*' <no-asterisk>+ '*'
    <bolded>        =:: '**' <no-asterisk>+ '**'
    <no-asterisk>   =:: '\*' | (Any character except '*')
    <underlined>    =:: '_' <no-underscore>+ '_'
    <no-underscore> =:: '\_' | (Any character except '_')
    <linked>      =:: '[' <link-text> '](' <link-target> ')'
    <link-text>   =:: '\]' | (Any character except ']')
    <link-target>     =:: '\)' | (Any character except ')')
    <color>           =:: '@@' <no-at> '@@' | '@@(' <color-code> ')' <no-at> '@@'
    <no-at>           =:: '\@' | (Any character except '@')
    <color-code>      =:: <color-number> | <color-name> |
    <color-number>    =:: <decimal-digit> | <decimal-digit> <decimal-digit>
    <color-name>      =:: (Any name from the table above, case-insentive)
    <code>            =:: '`' <no-backtick>+ '`' | '```' <no-triple-tick>+ '```'
    <no-backtick>     =:: '\`' | (Any character except '`')
    <no-triple-tick>  =:: '`' (Any character except '`')+ | '``' (Any character except '`')+ | (Any character except '`')

