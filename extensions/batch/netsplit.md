NETSPLIT and NETJOIN Batch Types
================================

Copyright (C) 2014 Alex Iadicicco <http://github.com/aji>

Unlimited redistribution and modification is allowed provided that the
above copyright notice and this permission notice remains intact.

NETSPLIT and NETJOIN Batch Types
--------------------------------

This document describes the format of the NETSPLIT and NETJOIN batch
types.

When a netsplit occurs, the server MUST put all resulting QUITs into
a single NETSPLIT batch. Similiarly, all netjoin-related JOINs MUST be
put into a *single* NETJOIN batch. Both types have 2 arguments, which are
the names of the servers that are splitting or joining, or *.net *.split
and *.net *.join if the server has chosen to hide links.

Clients that do not understand the NETSPLIT and NETJOIN batch subcommands
can safely interpret the batched QUITs and JOINs as standard QUITs
and JOINs.

Example
-------

An example netsplit is as follows:

    :irc.host BATCH +yXNAbvnRHTRBv NETSPLIT irc.hub other.host
    @batch=yXNAbvnRHTRBv :aji!a@a QUIT :irc.hub other.host
    @batch=yXNAbvnRHTRBv :nenolod!a@a QUIT :irc.hub other.host
    @batch=yXNAbvnRHTRBv :jilles!a@a QUIT :irc.hub other.host
    :irc.host BATCH -yXNAbvnRHTRBv

An example netjoin is as follows:

    :irc.host BATCH +4lMeQwsaOMs6s NETJOIN irc.hub other.host
    @batch=4lMeQwsaOMs6s :aji!a@a JOIN #atheme
    @batch=4lMeQwsaOMs6s :nenolod!a@a JOIN #atheme
    @batch=4lMeQwsaOMs6s :jilles!a@a JOIN #atheme
    @batch=4lMeQwsaOMs6s :nenolod!a@a JOIN #ircv3
    @batch=4lMeQwsaOMs6s :jilles!a@a JOIN #ircv3
    @batch=4lMeQwsaOMs6s :Elizacat!a@a JOIN #ircv3
    :irc.host BATCH -4lMeQwsaOMs6s
