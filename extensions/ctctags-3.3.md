---
title: IRCv3.3 `ctctags` Extension
layout: spec
copyrights:
  -
    name: "Nathaniel Filardo"
    period: "2015"
    email: "nwfilardo@gmail.com"
---

This document defines a new sub-class of IRC message-tags, so-called
"client-to-client" message-tags.  Syntactically, these are tag names whose
key

* is composed entirely of capital ASCII letters, numbers, and/or hyphens;
  and
* contains at least one capital letter

The `ctctags` capability causes the server to forward any such
client-to-client message-tags sent by other clients on `PRIVMSG` commands
routed to this client.

## Standardized client-to-client tags

As with all message-tags, client-to-client message-tags may be standardized
by the IRCv3 working group.  This document, by way of motivating example,
also serves to standardize one.

### `SUBJ` -- Message Subject Label

Some chat systems -- notably Zephyr and its derivatives such as Zulip --
include a notion of "instance", which can be thought of as a "sub-channel"
or a human-meaningful thread identifier. The `SUBJ` tag is designed to fill
this role for IRC; it is defined to carry a human-readable subject line,
suitable for use by clients for grouping or selective display of messages.
Clients are encouraged to provide mechanisms for sending a particular
message with a given `SUBJ` tag and setting default tag(s) for all messages
sent (e.g., per channel).

The `SUBJ` tag value is by fiat limited in length to 128 bytes of UTF-8 (so
as to not unduely crowd out other tags); servers SHOULD enforce this limit
and clients MUST NOT originate longer annotations though they MAY choose to
accept them.

## Example exchange

Assuming `clientA` and `clientB` have negotiated the use of the `ctctags`
extension while `clientC` has not and all three have joined `#channel`,

    clientA->server: @SUBJ=weather PRIVMSG #channel :What charming storms!
    server->clientB: @SUBJ=weather :clientA!ident@example.com PRIVMSG #channel :What charming storms!
    server->clientC: :clientA!ident@example.com PRIVMSG #channel :What charming storms!
