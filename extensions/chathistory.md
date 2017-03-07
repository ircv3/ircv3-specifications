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
## Notes for implementing work-in-progress version
This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `CHATHISTORY` command. Instead, implementations SHOULD use the `draft/CHATHISTORY` command to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed command.

## Description
This document describes the format of the `chathistory` extension. The `chathistory` extension uses the [chathistory][batch/chathistory] batch type, adding a client-side command for requesting `chathistory` content from the server.

The `chathistory` extension adds an optional `draft/msgid` to the `chathistory` batch reply.

Clients MUST support the [`batch`][batch], [`server-time`][server-time], and [`draft/labeled-response`][draft/labeled-response] capabilities. Clients SHOULD support the [`draft/msgid`][draft/msgid] capability.

When a client with the above mentioned capabilities requests `chathistory` content from the server (using the `CHATHISTORY` command outlined below), the server should return to the client a single batch containing a number of desired raw IRC lines equal to the `message_count` parameter specified, ending directly before the given timestamp for a negative `message_count`, after the number of messages specified for a positive `message_count`, or with the message directly proceeding the one with the specified `draft/msgid`. The raw IRC lines should be formatted and returned to the client as they were originally, with the addition of the above capability tags.

The `server-time` should be the time at which the message was originally sent and the `batch id` a unique ID to the entire batch. `draft/label` SHOULD be inclued and MUST be a unique ID used to identify the `chathistory` request and any replies. `draft/msgid` SHOULD be the `draft/msgid` originally sent with the message.

### `CHATHISTORY` Command
Chathistory content can be requested by the client to the server by sending the `CHATHISTORY` command to the server. A `batch` must be returned by the server. If no content exists to return, an empty batch should be returned to avoid the client waiting for a reply. Command support is sent to the client as the RPL_ISUPPORT 005 numeric `:irc.host 005 nick CHATHISTORY=max_message_count :are supported by this server`

Both the `message_count` and `max_message_count` MUST be non-zero integers and `max_message_count` MUST be positive. The client should not request a `message_count` with an absolute value greater than the `max_message_count` parameter sent in the command. If the `message_count` absolute value exceeds the `max_message_count`, server should return a number of lines equal to the `max_message_count` and the appropriate warning as described below.

#### Format
The `chathistory` content can requested using timestamps:

    @draft/label=ID CHATHISTORY target timestamp=YYYY-MM-DDThh:mm:ss.sssZ message_count

Alternatively, content can be requested using a `draft/msgid`:

    @draft/label=ID CHATHISTORY target draft/msgid=ID message_count

If `message_count` is positive, content MUST be retrieved from after the specifie timestamp or `draft/msgid`. If the `message_count` is negative, content MUST be retrieved from before the specified timestamp or `draft/msgid`.

#### Errors and Warnings
If the server receives an improperly formatted `CHATHISTORY` command, the `CMD_INVALID` error code should be returned.

If the absolute value of `message_count` exceeds the `max_message_count`, warn code `MAX_MSG_COUNT_EXCEEDED` should be returned. The command should continue to be processed as described above.

If no `chathistory` exists to return, the server should return the appropriate error code. `ACCESS_DENIED` should be sent if the user requests content they do not have permission to view.

### Examples
The examples below are written with `draft/msgid` and `draft/label` tags included. These tags are recommended.

#### Begin
    @draft/label=ID :irc.host BATCH +ID chathistory target
#### PRIVMSG
    @batch=ID;draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
#### NOTICE
    @batch=ID;draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host NOTICE target :message
#### End
    :irc.host BATCH -ID
#### Error
    @draft/label=ID :nick!ident@host CHATHISTORY ERR ERROR_CODE
#### Warning
    @draft/label=ID :nick!ident@host CHATHISTORY WARN WARN_CODE

## Use Cases
The batch type and supporting command are useful for allowing an "infinite scroll" type capability within the client. A client will, upon scrolling to the top of the active window or a manual trigger, may request `chathistory` from the server and, after receiving returned content, append it to the top of the window. Users can repeat this historic scrolling to retrieve prior history until limitations are met (see below).

## Limitations
Logging of messages and other actions must be enabled server-side and can be stored in any format desired, given appropriate software exists to retrieve and format such stored contnet. Scrollback can only be retrieved as far back as logs exist for the requesting user in the specified channel or query.

A method for securely identifying the requesting user must exist to ensure content is sent only to the appropriate users and clients. See below for more information.

## Security Considerations
Secure identification of users and clients must exist in order to ensure that users cannot obtain history that does not belong to them. Use of account names, internal account identifiers, or certificate fingerprints should be strongly considered when matching content to users. The server must verify the current user's identity matches that of the desired content. This information is not sent as part of the `CHATHISTORY` command and must be validated via other means, such as those stated above. Access must be checked first and return an `ACCESS_DENIED` error as appropriate. If no `ACCESS_DENIED` error exists, the server can check for returnable content.

## Current Implementations
A ZNC-module implementation exists as [znc-chathistory](https://github.com/MuffinMedic/znc-chathistory). Client-side support for the ZNC module is in progress. Interest in supporting the batch type and command without ZNC has also been obtained, although specifics are not available at this time.

Note: This module may be out of date while the specification is reviewed and modified.

[batch]: http://ircv3.net/specs/extensions/batch-3.2.html
[batch/chathistory]: http://ircv3.net/specs/extensions/batch/chathistory-3.3.html
[server-time]: http://ircv3.net/specs/extensions/server-time-3.2.html
[draft/msgid]: https://github.com/ircv3/ircv3-specifications/pull/285
[draft/labeled-response]: https://github.com/ircv3/ircv3-specifications/pull/162
