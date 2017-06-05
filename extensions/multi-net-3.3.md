# multi-net Capability Extension

Copyright (c) 2014 TrainerGuy22.

The `multi-net` capability would allow for clients to have different servers
handled through a single connection. This functionality is intended to bring
multiplexing capabilities to IRC bouncers, with rich client integration.

This will be achieved by simply adding a `net` tag to every message sent
between the client and the bouncer.

**Example** *(from the viewpoint of a bouncer)*:

    <-- @net=freenode :irc.freenode.net 001 TheMike :Welcome 
    <-- @net=freenode :irc.freenode.net 005 TheMike CHANTYPES=# 
    <-- @net=atheme :irc.atheme.net 001 TheMike :Welcome 
    <-- @net=atheme :irc.atheme.net 005 TheMike CHANTYPES=#&
    --> @net=freenode JOIN #znc <-- @net=freenode :TheMike!foo@bar JOIN #znc

This example would make the client respect two different servers, freenode
and athemenet, within the same connection.

For WHOIS/MSG/QUERY client commands (as well as similar commands) to work
correctly, clients would need to adopt a SERVER/USER for the nickname
parameter of the commands. If the SERVER part of the nickname is absent,
it would send the message with @net being the network that the currently
displayed channel/user is on. If the user is viewing a console, clients
should request they use the SERVER/USER syntax.

## Reasoning behind proposal

While working on IRC bouncer dartboard, we were working on implmenting
functionality for server multiplexing. We found that there was no currently
possible way to have nicknames for different servers. So, this extension
was originally born as prefixed-chan.

Issues were pointed out within the draft, so the draft was rewritten as
multi-net.
