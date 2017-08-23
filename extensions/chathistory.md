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

Clients MUST support the [`batch`][batch] and [`server-time`][server-time] capabilities. Clients SHOULD support the [`draft/labeled-response`][draft/labeled-response] and [`draft/message-tags`][draft/message-tags] capabilities.

When a client with the above mentioned capabilities requests `chathistory` content from the server (using the `draft/CHATHISTORY` command outlined below), the server SHOULD return to the client a single batch containing raw IRC lines starting with and exluduing the `start` parameter and up to and including the message at the `end` parameter. The raw IRC lines SHOULD be formatted and returned to the client as they were originally, with the addition of the above capability tags.

The `server-time` SHOULD be the time at which the message was received from the IRC server and the `batch id` a unique ID for the entire batch. `draft/label` SHOULD be included and MUST be a unique ID used to identify the `chathistory` request and any replies. A `draft/msgid` to identify each individual message MUST be the `draft/msgid` included when each message was first received from the IRC server.

### `CHATHISTORY` Command
`CHATHISTORY` content can be requested by the client to the server by sending the `draft/CHATHISTORY` command to the server. A `batch` MUST be returned by the server. If no content exists to return, an empty batch SHOULD be returned to avoid the client waiting for a reply. Command support is sent to the client as the RPL_ISUPPORT 005 numeric `:irc.host 005 nick draft/CHATHISTORY=max_message_count :are supported by this server`

Both the `message_count` and `max_message_count` MUST be integers. `message_count` MUST be a non-zero and `max_message_count` MUST be greater than or equal to zero. The client SHOULD not request a `message_count` with an absolute value greater than the `max_message_count` parameter sent in the command.

Both the `start` and `end` parameters support `draft/msgid` and `timestamp`. The `end` parameter also accepts a `message_count`. If the number of lines between the `start` and `end` parameters exceeds the `max_message_count`, the server SHOULD return a number of lines equal to the `max_message_count` and the appropriate warning as described below. A `max_message_count` of 0 indicates that no maximum exists.

A positive `message_count` or tag `draft/msgid` prepended with a `+` indicates the client is requesting content after the `start`. A negative  `message_count` or `draft/msgid` tag prepended with a `-` indicates the client is requested content before the specified `start`.

The `target` parameter specifies a single channel or query from which history SHOULD be retrieved. Wildcards or multiple targets are not supported.

#### Format
`chathistory` uses the following generic format:

    @draft/label=ID draft/CHATHISTORY <target> <start> <end>

The `chathistory` content can requested using timestamps:

    @draft/label=ID draft/CHATHISTORY <target> timestamp=YYYY-MM-DDThh:mm:ss.sssZ message_count=<message_count>

Alternatively, content can be requested using a `draft/msgid`:

    @draft/label=ID draftCHATHISTORY <target> draft/msgid=<message_id> timestamp=<timestamp>

Content can also be requested up to a specified timestamp or `draft/msgid` in place of the `message_count`. The start and end parameter types do not have to match:

    @draft/label=ID draftCHATHISTORY <target> timestamp=<timestamp> +draft/msgid=<message_id>   

#### Errors and Warnings
If the server receives a`draft/CHATHISTORY` command with missing parameters, the `ERR_NEEDMOREPARAMS` error code SHOULD be returned.

If the number of lines between the `start` and `end` parameters exceeds the `max_message_count`, warn code `MAX_MSG_COUNT_EXCEEDED` SHOULD be returned. The command SHOULD continue to be processed as described above.

If the target has no `chathistory` content to return or the user does not have permission to view the requested content, `ERR_NOSUCHNICK` or `ERR_NOSUCHCHANNEL` SHOULD be returned accordingly.

### Examples
The examples below are written with `draft/msgid` and `draft/label` tags included. These tags are recommended.

#### Begin
    @draft/label=ID :irc.host BATCH +ID chathistory target
#### PRIVMSG
    @batch=ID;draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :message
#### NOTICE
    @batch=ID;draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host NOTICE target :message
#### ACTION
    @batch=ID;draft/msgid=ID;time=YYYY-MM-DDThh:mm:ss.sssZ :nick!ident@host PRIVMSG target :ACTION message
#### End
    :irc.host BATCH -ID
#### Error
    @draft/label=ID :nick!ident@host draft/CHATHISTORY ERR ERROR_CODE
#### Warning
    @draft/label=ID :nick!ident@host draft/CHATHISTORY WARN WARN_CODE

## Use Cases
The batch type and supporting command are useful for allowing an "infinite scroll" type capability within the client. A client will, upon scrolling to the top of the active window or a manual trigger, may request `chathistory` from the server and, after receiving returned content, append it to the top of the window. Users can repeat this historic scrolling to retrieve prior history until limitations are met (see below).

## Limitations
Logging of messages and other actions MUST be enabled server-side and can be stored in any format desired, given appropriate software exists to retrieve and format such stored contnet. Scrollback can only be retrieved as far back as logs exist for the requesting user in the specified channel or query.

A method for securely identifying the requesting user MUST exist to ensure content is sent only to the appropriate users and clients. See below for more information.

## Security Considerations
Secure identification of users and clients MUST exist in order to ensure that users cannot obtain history they are not authorized to view. Use of account names, internal account identifiers, or certificate fingerprints SHOULD be strongly considered when matching content to users. The server MUST verify the current user's identity matches that of the desired content. This information is not sent as part of the `draft/CHATHISTORY` command and MUST be validated via other means, such as those stated above. Access MUST be checked first and return an `ERR_NOSUCHNICK` or `ERR_NOSUCHCHANNEL` error as appropriate. If no authorization error exists, the server can check for returnable content. If no returntable content is found, the server MUST send an `ERR_NOTEXTTOSEND` error. The server MUST NOT expose the existence of valid targets to unauthorized users.

While a `max_message_count` of 0 MAY be used to indicate no limit exists, servers SHOULD set and enforce a reasonable `max_message_count` and properly throttle `draft/CHATHISTORY` commands to prevent abuse.

## Current Implementations
A ZNC-module implementation exists as [znc-chathistory](https://github.com/MuffinMedic/znc-chathistory). Client-side support for the ZNC module is in progress. Interest in supporting the batch type and command without ZNC has also been obtained, although specifics are not available at this time.

Note: This module may be out of date while the specification is reviewed and modified.

[batch]: http://ircv3.net/specs/extensions/batch-3.2.html
[batch/chathistory]: http://ircv3.net/specs/extensions/batch/chathistory-3.3.html
[server-time]: http://ircv3.net/specs/extensions/server-time-3.2.html
[draft/message-tags]: http://ircv3.net/specs/extensions/message-ids.html
[draft/labeled-response]: http://ircv3.net/specs/extensions/labeled-response.html
