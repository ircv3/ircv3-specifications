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
This document describes the format of the `chathistory` extension. The `chathistory` extension uses the [chathistory][batch/chathistory] batch type, adding a client-side command for requesting `chathistory` content from the server.

The `chathistory` extension adds an optional `draft/msgid` to the `chathistory` batch reply.

Clients MUST support the [`batch`][batch] and [`server-time`][server-time] capabilities. Clients SHOULD support the [`draft/labeled-response`][draft/labeled-response] and [`draft/message-tags`][draft/message-tags] capabilities.

When a client with the above mentioned capabilities requests `chathistory` content from the server (using the `CHATHISTORY` command outlined below), the server SHOULD return to the client a single batch containing raw IRC lines starting with and exluduing the `start` parameter and up to and including the message at the `end` parameter. The raw IRC lines SHOULD be formatted and returned to the client as they were originally, with the addition of the above capability tags.

The `server-time` SHOULD be the time at which the message was received from the IRC server and the `batch id` a unique ID for the entire batch. `draft/label` SHOULD be included and MUST be a unique ID used to identify the `chathistory` request and any replies. A `draft/msgid` to identify each individual message MUST be the `draft/msgid` included when each message was first received from the IRC server.

### `CHATHISTORY` Command
`CHATHISTORY` content can be requested by the client to the server by sending the `CHATHISTORY` command to the server. A `batch` MUST be returned by the server. If no content exists to return, an empty batch SHOULD be returned to avoid the client waiting for a reply. Command support is sent to the client as the RPL_ISUPPORT 005 numeric `:irc.host 005 nick CHATHISTORY=max_messages :are supported by this server`

Both the `limit` and `max_messages` MUST be integers, and `max_messages` MUST be greater than or equal to zero. The client SHOULD NOT request a `limit` with an absolute value greater than the `max_messages` parameter sent in the command. If the `limit` is 0 or no `limit` is provided, the number of messages equal to `max_messages` MUST BE returned.

Both the `start` and `end` parameters support `draft/msgid` and `timestamp`. If the number of lines between the `start` and `end` parameters exceeds the `max_messages`, the server SHOULD return a number of lines equal to the `max_messages` and the appropriate warning as described below. A`limit` or `max_messages` of 0 indicates that no limit or maximum exists.

The `target` parameter specifies a single channel or query from which history SHOULD be retrieved. Wildcards or multiple targets are not supported.

#### Subcommand

The following subcommands are used to describe how the server should return messages relative to the `timestamp(s)` or `draft/msgid(s)` given.

`BEFORE` get up to `limit` number of messages before the given `timestamp` or `draft/msgid`. The `limit` MUST BE positive.

`AFTER` get up to `limit` number of messages after the given `timestamp` or `draft/msgid`. The `limit` MUST BE positive.

`LATEST` get the most recent (up to `limit`) messages that have been sent since the given `timestamp` or `draft/msgid`. The `limit` MUST BE positive.

`AROUND` get a number of messages before and after the `timestamp` or `draft/msgid`. The `limit` MUST BE positive.

`BETWEEN` get up to `limit` number of messages between the given `timestamps` or `draft/msgids`. The `limit` MAY positive or negative. If the `limit` is positive, messages will be returned in ascending order starting at the oldest message. If the `limit` is negative, messages will be returned in decending order starting at the newest message. 

#### Format
`chathistory` uses the following generic format:

    @draft/label=ID CHATHISTORY <target> <subcommand> <start> <end> [<direction>] [<limit>]

The `chathistory` content can requested using timestamps:

    @draft/label=ID CHATHISTORY <target> <subcommand> timestamp=YYYY-MM-DDThh:mm:ss.sssZ [<direction>] [<limit>]

Alternatively, content can be requested using a `draft/msgid`:

    @draft/label=ID CHATHISTORY <target> <subcommand> draft/msgid=<message_id> timestamp=<timestamp>

For `BEFORE`, `AFTER`, `LATEST`, and `AROUND`, a single `timestamp` or `draft/msgid` is required.  For `BETWEEN`, both a `start` and `end` `timestamp` and/or `draft/msgid` is required. The  `limit` is optional for all subcommands.

#### Errors and Warnings
If the server receives a `CHATHISTORY` command with missing parameters, the `NEED_MORE_PARAMS` error code SHOULD be returned.

If the number of lines between the `start` and `end` parameters exceeds the `max_messages`, warn code `MAX_MESSAGES_EXCEEDED` SHOULD be returned. The command SHOULD continue to be processed as described above.

If the target has no `chathistory` content to return or the user does not have permission to view the requested content, `NO_SUCH_NICK` or `NO_SUCH_CHANNEL` SHOULD be returned accordingly.

### Examples
The examples below are written with `draft/msgid` and `draft/label` tags included. These tags are recommended.

    @draft/label=ID :irc.host BATCH +ID chathistory target

    @batch=ID;draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message

    @batch=ID;draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host NOTICE target :message

    @batch=ID;draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :ACTION message

    :irc.host BATCH -ID

    @draft/label=ID :nick!ident@host ERR CHATHISTORY ERR_CODE

    @draft/label=ID :nick!ident@host WARN CHATHISTORY WARN_CODE

## Use Cases
The batch type and supporting command are useful for allowing an "infinite scroll" type capability within the client. A client will, upon scrolling to the top of the active window or a manual trigger, may request `chathistory` from the server and, after receiving returned content, append it to the top of the window. Users can repeat this historic scrolling to retrieve prior history until limitations are met (see below).

## Limitations
Logging of messages and other actions MUST be enabled server-side and can be stored in any format desired, given appropriate software exists to retrieve and format such stored contnet. Scrollback can only be retrieved as far back as logs exist for the requesting user in the specified channel or query.

A method for securely identifying the requesting user MUST exist to ensure content is sent only to the appropriate users and clients. See below for more information.

## Security Considerations
Secure identification of users and clients MUST exist in order to ensure that users cannot obtain history they are not authorized to view. Use of account names, internal account identifiers, or certificate fingerprints SHOULD be strongly considered when matching content to users. The server MUST verify the current user's identity matches that of the desired content. This information is not sent as part of the `CHATHISTORY` command and MUST be validated via other means, such as those stated above. Access MUST be checked first and return a `NO_SUCH_NICK` or `NO_SUCH_CHANNEL` error as appropriate. If no authorization error exists, the server can check for returnable content. If no returntable content is found, the server MUST send a `NO_TEXT_TO_SEND` error. The server MUST NOT expose the existence of valid targets to unauthorized users.

While a `max_messages` of 0 MAY be used to indicate no limit exists, servers SHOULD set and enforce a reasonable `max_messages` and properly throttle `CHATHISTORY` commands to prevent abuse.

## Current Implementations
A ZNC-module implementation exists as [znc-chathistory](https://github.com/MuffinMedic/znc-chathistory). Client-side support for the ZNC module is in progress. Interest in supporting the batch type and command without ZNC has also been obtained, although specifics are not available at this time.

Note: This module may be out of date while the specification is reviewed and modified.

[batch]: http://ircv3.net/specs/extensions/batch-3.2.html
[batch/chathistory]: http://ircv3.net/specs/extensions/batch/chathistory-3.3.html
[server-time]: http://ircv3.net/specs/extensions/server-time-3.2.html
[draft/message-tags]: http://ircv3.net/specs/extensions/message-ids.html
[draft/labeled-response]: http://ircv3.net/specs/extensions/labeled-response.html