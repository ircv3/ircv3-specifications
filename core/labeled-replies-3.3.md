---
title: IRCv3.3 Labeled replies
layout: spec
copyrights:
  -
    name: "Alexey Sokolov"
    period: "2015"
    email: "alexey-irc@asokolov.org"
---
Name of this capability is `labeled-replies`.
If client requests `labeled-replies`, it MUST also request `batch` capability.

If the capability is enabled, client MAY tag its messages with `label` tag.

For messages from client which are tagged with `label`, server MUST send exactly
one message tagged with the same `label` value in response to such message.
If response contains more than one message, `batch` can be used to group them.
In that case start of the batch MUST be tagged with the label, and the batch type MUST be `labeled-response`.

If the message doesn't require any response, an empty batch MUST be sent.

## Examples

1. `echo-message`

    ```
    Client: @label=pQraCjj82e PRIVMSG #channel :\x02Hello!\x02
    Server: @label=pQraCjj82e :nick!user@host PRIVMSG #channel :Hello!
    ```

2. Failed `PRIVMSG` with `ERR_NOSUCHNICK`

    ```
    Client: @label=dc11f13f11 PRIVMSG nick :Hello
    Server: @label=dc11f13f11 :irc.example.com 401 * nick :No such nick/channel
    ```
    
3. `WHOIS`

    ```
    Client: @label=mGhe5V7RTV WHOIS nick
    Server: @label=mGhe5V7RTV :irc.example.com BATCH +NMzYSq45x labeled-response
    Server: @batch=NMzYSq45x :irc.example.com 311 client nick ~ident host * :Name
    ...
    Server: @batch=NMzYSq45x :irc.example.com 318 client nick :End of /WHOIS list.
    Server: :irc.example.com BATCH -NMzYSq45x
    ```
