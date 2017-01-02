---
title: IRCv3 scrollback Batch Type
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Evan Magaliff"
    period: "2017"
    email: "evan@muffinmedic.net"
---
## Description
### `scrollback` Batch Type
This document describes the format of the `scrollback` batch type. The `scrollback` batch type is an extension of the `chathistory` batch type, adding additional IRC events to it's output and implementing a client-side command for requesting scrollback content from the server.

The `scrollback` batch type takes a single target parameter, which must be either a channel or query name.

Client side support for the [`batch`][batch], [`server-time`][server-time], and [`draft/msgid`](https://github.com/ircv3/ircv3-specifications/pull/285) capabilities is required.

When a client with the above mentioned capabilities requests scrollback content from the server (using the `scrollback` command outlined below), the server should return to the client a single batch containing a number of desired raw IRC lines equal to the `message_count` parameter specified, beginning with the first message previous to the last-known timestamp. The raw IRC lines are to be formatted and returned to the client as they would be originally, with the addition of the above capability tags.

The `server-time` should be the time at which the message was originally sent, the `batch id` a randomly generated string unique to the entire batch, and the `draft/msgid` a unique identifier for each message.

### `scrollback` Command
Scrollback content can be requested by the client to the server by sending the `SCROLLBACK` command to the server. No acknowledgement by the server of the command is required other then returning the requested content. Command support is sent to the client as the RPL_ISUPPORT 005 numeric `:irc.host 005 nick SCROLLBACK :are supported by this server` 

#### Format
    SCROLLBACK target YYYY-MM-DDThh:mm:ss.sssZ message_count
    SCROLLBACK #channel 2016-11-19T18:02:01.000Z 50

If no message_count is known, `*` can be used as default

## Examples

### Begin
    :irc.host BATCH +ID muffinmedic.net/scrollback target
    :irc.host BATCH +XNyDSitp9MvcX scrollback #channel
### PRIVMSG
    @batch=ID;draft/msgid=UUID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
    @batch=XNyDSitp9MvcX;draft/msgid=;time=2016-11-19T18:02:01.001Z :foo!bar@example.com PRIVMSG #channel :The ZNC SCROLLBACK command is going to be fantastic!
### NOTICE
    @batch=ID;draft/msgid=UUID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host NTOICE target :message
    @batch=XNyDSitp9MvcX;draft/msgid=;time=2016-11-19T18:02:02.002Z :foo!bar@example.com NOTICE #channel :Announcing the new ZNC SCROLLBACK command!
### JOIN
    @batch=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host JOIN :#channel
    @batch=XNyDSitp9MvcX;draft/msgid=;time=2016-11-19T18:02:03.003Z :foo!bar@example.com JOIN :#channel
### PART
    @batch=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PART #channel :reason
    @batch=XNyDSitp9MvcX;draft/msgid=;time=2016-11-19T18:02:04.004Z :foo!bar@example.com PART #channel :This place is too addicting
### QUIT
    @batch=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host QUIT #channel :reason
    @batch=XNyDSitp9MvcX;draft/msgid=;time=2016-11-19T18:02:05.005Z :foo!bar@example.com QUIT #channel :Restarting in debug mode
### KICK
    @batch=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :op_nick!ident@host KICK #channel kicked_nick :message
    @batch=XNyDSitp9MvcX;draft/msgid=;time=2016-11-19T18:02:06.006Z :foo!bar@example.com KICK #channel CupcakeMedic :Muffins > cupcakes
### NICK
    @batch=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :old_nick!ident@host NICK :new_nick
    @batch=XNyDSitp9MvcX;draft/msgid=;time=2016-11-19T18:02:07.007Z :foo!bar@example.com NICK :Evan
### TOPIC
    @batch=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host TOPIC #channel :topic
    @batch=XNyDSitp9MvcX;draft/msgid=;time=2016-11-19T18:02:08.008Z :foo!bar@example.com TOPIC #channel :Check out the new ZNC SCROLLBACK command
### MODE
    @batch=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :op_nick!ident@host MODE #channel mode(s) parameter(s)
    @batch=XNyDSitp9MvcX;draft/msgid=;time=2016-11-19T18:02:09.009Z :foo!bar@example.com MODE #channel +b *!*@example.com
### End
    :irc.host BATCH -ID
    :irc.host BATCH -XNyDSitp9MvcX

## Use cases
The batch type and supporting command are useful for allowing an "infinite scroll" type capability within the client itself. A client will, upon scrolling to the top of the active window, request scrollback from the server and, upon receiving such content, append it to the top of the window. Users can repeat this historic scrolling to retrieve prior history until limitations are met (see below).

## Limitations
Logging of messages and other actions must be enabled server-side and can be stored in any format desired, given appropriate software exists to retrieve and format such stored contnet. Scrollback can only be retrieved as far back as logs exist for the requesting user in the specified channel or query.

A method for securely identifying the requesting user must exist to ensure content is sent only to the appropriate users and clients. See below for more information.

## Security considerations
Secure identification of users and clients must exist in order to ensure that users cannot obtain history that does not belong to them. Use of account names, internal account identifiers, or certificate fingerprints should be strongly considered when matching content to users. The server must verify the current user's identity matches that of the desired content. This information is not sent as part of the `SCROLLBACK` command and must be validated via other means, such as those stated above.

## Current Implementations
A ZNC-module implementation exists as [znc-scrollback](https://github.com/MuffinMedic/znc-scrollback). Client-side support for the ZNC module is in progress. Interest in supporting the batch type and command without ZNC has also been obtained, although specifics are not available at this time.

[batch]: ../batch-3.2.html
[chathistory]: chathistory-3.3.html
[server-time]: ../server-time-3.2.html
