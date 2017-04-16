---
title: IRCv3.3 Message Data framework
layout: spec
copyrights:
  -
    name: "James Wheare"
    period: "2015"
    email: "james@wheare.org"
---
This specification concerns adding an optional out-of-band data tag to messages
using the [message tagging](../core/message-tags-3.2.html) framework, and a
mechanism for defining such tags using the [metadata](../core/metadata-3.2.html)
framework.

Data tags allow a client to attach custom structured key value data to a message
that needn't be negotiated beforehand with a server.

## The `message-data` capability

A client MUST request the `message-data` capability in order to receive data
tags.

## The `data-` prefixed message tag

To send a data tagged message with the key `foo` and the value `bar`, we send the
following:

    @data-foo=bar PRIVMSG #ircv3 :hello #channel

The IRC server would then attach all `data-` prefixed tags to the message when
it distributes it to clients supporting the `message-data` capability.

Vendor prefixed tags must also be supported, in the form:
    
    <vendor>/data-<key>

## Defining a data schema

In order to provide a path for standardisation of data tags, a client SHOULD set
metadata defining its use of data tags, using the `message-data` metadata key
and a set of space-separated values.

A client supporting only one set of data tags:

    METADATA * SET message-data :example.com/message-data

A client suppoting more than one set of data tags:

    METADATA * SET message-data :example.com/message-data example.com/extra-data

Specifying the tagset in this way allows clients to interpret them according to
specifications provided out of band.

To avoid having to query metadata for each data-tag received, the `metadata-notify` capability SHOULD be requested to allow data tag definitions to be discovered automatically.

### Non-normative

The following examples describe how an implementation might use certain
features. Unlike previous examples, they are in no way intended to guide
implementations' behaviour.

A client with nick `Bot` that provides title fetching services for links sent in
a channel can send additional data including the favicon of the fetched site by
first setting metadata on connect:
    
    METADATA * SET message-data :example.com/favicon-data-tag

And then send tagged messages to the channel:
    
    @data-favicon=https://example.com/icon.png :Bot!~Bot_user@example.com PRIVMSG #channel :Example title

METADATA GET on `Bot` would return

    METADATA Bot GET message-data
    :irc.example.com 761 Bot message-data * :example.com/favicon-data-tag
    :irc.example.com 762 :end of metadata

Metadata notifications for `Bot` would include:

    :irc.example.com METADATA Bot message-data * :example.com/favicon-data-tag
