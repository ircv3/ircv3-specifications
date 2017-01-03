---
title: IRCv3 chathistory extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Evan Magaliff"
    period: "2017"
    email: "evan@muffinmedic.net"
---
## Description
This document describes the format of the `chathistory` extension. The `chathistory` extension uses the `chathistory` batch type, adding a client-side command for requesting chathistory content from the server.

The `chathistory` extension adds an optional `draft/msgid` to the `chathistory` batch reply.

Client side support for the [`batch`][batch] and [`server-time`][server-time] capabilities is required. Support for the [`draft/msgid`](https://github.com/ircv3/ircv3-specifications/pull/285) capability is optional but recommended.

When a client with the above mentioned capabilities requests chathistory content from the server (using the `CHATHISTORY` command outlined below), the server should return to the client a single batch containing a number of desired raw IRC lines equal to the `message_count` parameter specified, ending directly before the given timestamp or with the message directly proceeding the one with the specified `draft/msgid`. The raw IRC lines should be formatted and returned to the client as they were originally, with the addition of the above capability tags.

The `server-time` should be the time at which the message was originally sent, the `batch id` a randomly generated string unique to the entire batch, and the optional `draft/msgid`, if included, the `draft/msgid` originally sent with the message.

### `CHATHISTORY` Command
Chathistory content can be requested by the client to the server by sending the `CHATHISTORY` command to the server. A `batch` must be returned by the server. If no content exists to return, an empty batch should be returned to avoid the client waiting for a reply. Command support is sent to the client as the RPL_ISUPPORT 005 numeric `:irc.host 005 nick CHATHISTORY=max_message_count :are supported by this server`

#### Format
The `chathistory` content can requested using timestamps:

    CHATHISTORY target timestamp=YYYY-MM-DDThh:mm:ss.sssZ message_count
    CHATHISTORY #channel timestamp=2016-11-19T18:02:01.000Z 50

Alternatively, content can be requested using a `draft/msgid`:

    CHATHISTORY target draft/msgid=ID message_count
    CHATHISTORY #channel draft/msgid=774ba1b6-202b-448c-b23a-6150ce5681fd 50

If no message_count is known, `*` should be used to specify the default value.

## Examples
The examples below are written with `draft/msgid` tags included. This tag is optional but recommended.

### Begin
    :irc.host BATCH +ID chathistory target
    :irc.host BATCH +XNyDSitp9MvcX chathistory #channel
### PRIVMSG
    @batch=ID;draft/msgid=UUID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    @batch=XNyDSitp9MvcX;draft/msgid=eb703092-0782-4c28-bf7c-f9e5c45963ca;time=2016-11-19T18:02:01.001Z :foo!bar@example.com PRIVMSG #channel :The CHATHISTORY specification is going to be fantastic!
### NOTICE
    @batch=ID;draft/msgid=UUID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host NOTICE target :message
    @batch=XNyDSitp9MvcX;draft/msgid=cb302078-ce4e-4866-82b8-480ec58094b3;time=2016-11-19T18:02:02.002Z :foo!bar@example.com NOTICE #channel :Announcing the new IRCv3 CHATHISTORY extension draft!
### End
    :irc.host BATCH -ID
    :irc.host BATCH -XNyDSitp9MvcX

## Use cases
The batch type and supporting command are useful for allowing an "infinite scroll" type capability within the client itself. A client will, upon scrolling to the top of the active window, request chathistory from the server and, upon receiving such content, append it to the top of the window. Users can repeat this historic scrolling to retrieve prior history until limitations are met (see below).

## Limitations
Logging of messages and other actions must be enabled server-side and can be stored in any format desired, given appropriate software exists to retrieve and format such stored contnet. Scrollback can only be retrieved as far back as logs exist for the requesting user in the specified channel or query.

A method for securely identifying the requesting user must exist to ensure content is sent only to the appropriate users and clients. See below for more information.

## Security considerations
Secure identification of users and clients must exist in order to ensure that users cannot obtain history that does not belong to them. Use of account names, internal account identifiers, or certificate fingerprints should be strongly considered when matching content to users. The server must verify the current user's identity matches that of the desired content. This information is not sent as part of the `CHATHISTORY` command and must be validated via other means, such as those stated above.

## Current Implementations
A ZNC-module implementation exists as [znc-chathistory](https://github.com/MuffinMedic/znc-chathistory). Client-side support for the ZNC module is in progress. Interest in supporting the batch type and command without ZNC has also been obtained, although specifics are not available at this time.

[batch]: http://ircv3.net/specs/extensions/batch-3.2.html
[chathistory]: http://ircv3.net/specs/extensions/batch/chathistory-3.3.html
[server-time]: http://ircv3.net/specs/extensions/server-time-3.2.html
