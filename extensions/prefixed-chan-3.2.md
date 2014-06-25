# prefixed-chan Capability Extension

Copyright (c) 2014 TrainerGuy22.

## Description

The `prefixed-chan` capability would allow for clients to associate different
nicknames with a channel prefix, effectively having multiple nicknames. While
not very useful for the ordinary client/server connection, this capability
will open up a new breed of IRC bouncer functionality.

This will be achieved by simply adding a `prefix` tag to the NICK message sent
by the server (or, as in this capability's intended functionality, the
bouncer).

**Example** *(from the viewpoint of a client)*:

    --> @prefix=atheme NICK foo

While this example dosen't make much sense given the usecase, it would make the client associate
itself with the nickname 'testnick' in any channel that starts with 'atheme'.

## Reasoning behind proposal

While working on IRC bouncer dartboard, we were working on implmenting our functionality for
having more than one server connected at once through a client. We found that there was no
possible way to have nicknames for different servers, in which their channels were prefixed
with the server name. So, this extension was born.
