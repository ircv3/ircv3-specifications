# IRCv3 Server Command Discovery

Copyright (c) 2015 Kythyria Tieran <kythyria@berigora.net>

Unlimited redistribution and modification of this document is allowed
provided that the above copyright notice and this permission notice
remains intact.

## Rationale

Many servers (and bouncers, which are servers from the point of view of a
client) implement non-standard commands which it is nevertheless entirely
reasonable for a client to expose in a user interface in some manner more
intelligent than passing unrecognised `/command`s on to the server.

For instance, ZNC's "detach" facility, an enormous number of the commands
in a typical services implementation, or commands only meaningful for one
network or rare server configuration, are unlikely to be standardised, or
be explicitly supported by clients.

This document specifies a way to list the commands a server supports in a
machine-readable, in-band manner.

## Indicating support

A server supporting this specification must announce `CMDLIST=<token>` in
its `RPL_ISUPPORT` banner. The `<token>` may be any valid string provided
that it changes whenever the reply to `LISTCMDS` does, such as a sequence
number or hash of the reply.

## Requesting a command list

The client sends a `LISTCMDS` message to the server, with no parameters.

The server responds with a sequence of `RPL_CMDLIST` messages, indicating
the end with `RPL_CMDLISTEND`.

## Format of `RPL_CMDLIST`

```
RPL_COMMANDLIST <command> <context> <parameter1> <parameter2> ...
```

If there are more than 13 parameters, they are to be placed as a space-separated
list in the final parameter, with a leading colon indicating the beginning of a
final parameter containing spaces:
```
RPL_COMMANDLIST FOO nick o:a o:b o:c o:d o:e o:f o:g o:h o:i o:j o:k o:l :o:m o:n :label:Frob 15 things
```

`command` is the verb used by this command (so, the same value might recur
if subcommands are involved).  
`context` is a hint as to in which context the command is useful. A client
can use this to determine where in a GUI, if anywhere, a command should be
displayed.

The following `context`s are defined:
```
channel     | Any channel
nick        | Any nick
nickchan    | Nick in a channel
target      | Nick or channel, or possibly a targetlist
self        | The user issuing the command
hetwork     | The network the user is connected to
other       | Something else
```

### Parameters
Parameters are described in the format
```
<type>:<options>:<name>
```
Where `<name>` is any text that does not contain spaces unless it is the final
parameter, and denotes the name of the parameter

The value of `<type>` indicates what kind of data this parameter takes, and the
`<options>` field, which may be empty, indicates flags. Presently two flags are
defined, `c` and `o`. `o` indicates a parameter is entirely optional, while `c`
indicates that the user should not be required to provide a value explicitly if
the client can determine it from the context in which the command is invoked.

The following types are defined:
```
sc          | Subcommand; literal text to include to invoke this one.
chan        | Channel name
nick        | Nickname
user        | Services username (which is usually a nick)
target      | Nick or channel, or possibly a targetlist.
pattern     | Pattern such as a wildcarded hostmask
o           | Some other parameter
text        | Long text (must be the last non-label parameter!)
label       | Not actually parameter. UI text to display for this command.
```

The last three are of somewhat special nature: `text` must be the last parameter
other than `label`, since spaces are permitted in this--the payload of, say,
`PRIVMSG` is a `text` parameter--and `label` does not define a parameter at all,
but rather suggests a name to use when displaying the command in a GUI, such as
if it is presented as a menu item.

## Example
Suppose `RPL_CMDLIST` and `RPL_CMDLISTEND` are `297` and `299` respectively.
```
S: 005 EXCEPTS INVEX CMDLIST=571
C: LISTCMDS

... snip ...

S: 298 PRIVMSG target target:c:destination text:message
S: 298 KICK nickchan chan:c:channel nick:c:nick text:reason
S: 298 MODE channel chan:c:channel sc:+b pattern:target label:Ban
S: 298 MODE nickchan chan:c:channel sc:+o nick:c:target :label:Give op
S: 298 MODE nickchan chan:c:channel sc:-o nick:c:target :label:Take op
S: 298 ZNC channel sc:detach chan:c:channel
S: 298 NS self sc:register o::password o::email :label:Register this nickname
S: 299 :End of command list.
```

This indicates, for instance, that sending `ZNC detach <channel>` is a valid
command, that `<channel>` should be inferred by looking at what the currently
selected channel is if the user doesn't specify explicitly, and that the proper
menu or other collection of actions to which the command should be added is the
one pertaining to channels.

It also indicates that sending `MODE <channel> +o <target>` performs an operation
labelled `Give op`, that this should be added to the list of operations that may
be performed on users in the same channel as oneself, and that `<channel>` and
`<target>` are readily inferred from context.

## Notes
If a command is presented as a GUI element, it is recommended that a client
nonetheless permit the user to enter it textually. 
