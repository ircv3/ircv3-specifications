---
title: IRCv3.3 `maxline` Extension
layout: spec
copyrights:
  -
    name: "Daniel Oaks"
    period: "2016"
    email: "daniel@danieloaks.net"
---
A method for negotiating longer protocol lines.

## Introduction

Currently, IRC lines are limited to 512 bytes. There has been much desire to allow longer lines while sending or receiving IRC traffic, allowing longer lines with `PRIVMSG`/`NOTICE`, and solving related issues that come into play while working with the IRC protocol. These issues in particular come into play as the protocol is extended to provide greater functionality.

The `maxline` capability specifies the maximum length of the tags, and of the rest of the message, that the server allows. As appropriate, lines will be truncated or split to account for clients which are still restricted to the standard (pre-maxline) limit.

## Architecture

### The `maxline` Capability

The `maxline` capability, when advertised, MUST have a value which is a positive integer. For example:

    C: CAP LS
    S: CAP * LS :maxline=2048 

Similarly to standard message handling, tags and the rest of the message have separate length values. The value of the `maxline` capability represents the maximum number of bytes that the tags section, and that the rest of the message, can take up. Line length calculation is done this way in order to better integrate with methods currently used by IRC software to limit line lengths.

As an example, if `maxline` is 1024 then the maximum size of a full IRC message would be 2048 bytes (1024 for the tags, 1024 for the rest of the message).

Servers MUST truncate incoming messages to their `maxline` values before processing them.

`maxline` MUST be at least 512 and SHOULD default to at least 2048.

### Interaction with PRIVMSG and NOTICE

If a client has negotiated the `maxline` capability and sends a `PRIVMSG` or a `NOTICE` message that is longer than 512 bytes, the receiving server MUST split this into multiple regular (512-byte) length messages when sending it to clients that have not negotiated the `maxline` capability.

Servers SHOULD split on whitespace, but may use whatever method is easiest for them to implement. Splitting does not need to occur at the exact max length of the message, and servers can instead opt to split a number of characters earlier to simplify processing. Lines SHOULD NOT be split in the middle of a UTF-8 character.

Servers MAY split other commands/numerics into multiple lines in a way similar to `PRIVMSG` and `NOTICE` above.

### The `truncated` Tag

The `truncated` tag, when present, indicates that a message has been truncated due to the client's line length. It may be sent to any client which supports message tags, as deemed appropriate by the IRCd.

As an example, this tag may be used when a long channel `TOPIC` is set, but cannot correctly be relayed to the given user due to its length.

### Examples

In the following examples, the default message length is presumed to be 100 bytes (rather than 512) and the extended length is presumed to be 400 bytes (rather than 2048). This is done purely to simplify presentation and convey the concept more easily.

* ` ->` conveys lines being sent from this client to the IRC server.
* `<- ` conveys lines being sent from the server to this client.

### Sending a long privmsg to someone else

In this example, C1 and C2 have negotiated `maxlen`.

    C1  -> PRIVMSG coolfriend :Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
    C2 <-  :c1!test@localhost PRIVMSG coolfriend :Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

In this example, C1 has negotiated `maxlen` but C2 has not.

    C1  -> PRIVMSG coolfriend :Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
    C2 <-  :c1!test@localhost PRIVMSG coolfriend :Lorem ipsum dolor sit amet, consectetur adipiscing elit,
    C2 <-  :c1!test@localhost PRIVMSG coolfriend :sed do eiusmod tempor incididunt ut labore et dolore
    C2 <-  :c1!test@localhost PRIVMSG coolfriend :magna aliqua.

### Setting a long topic and having another user run into it

In this example, C1 and C2 have negotiated `maxlen`.

    C1  -> TOPIC #coolchan :Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
    C1 <-  332 c1 #coolchan :Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

    C2  -> TOPIC #coolchan
    C2 <-  332 c1 #coolchan :Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

In this example, C1 has negotiated `maxlen` but C2 has not.

    C1  -> TOPIC #coolchan :Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
    C1 <-  332 c1 #coolchan :Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

    C2  -> TOPIC #coolchan
    C2 <-  @truncated 332 c1 #coolchan :Lorem ipsum dolor sit amet, consectetur adipiscing elit,
