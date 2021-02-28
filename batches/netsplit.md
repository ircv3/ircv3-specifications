---
title: "`netsplit` and `netjoin` Batch Types"
layout: spec
redirect_from:
  - /specs/extensions/batch/netsplit-3.2.html
copyrights:
  -
    name: "Alex Iadicicco"
    period: "2014"
    email: "http://github.com/aji"
---

## Description

This document describes the format of the `netsplit` and `netjoin` batch
types.

When a netsplit occurs, the server MUST put all resulting QUITs into
a single `netsplit` batch. Similarly, all netjoin-related JOINs MUST be
put into a *single* `netjoin` batch. Both types have 2 arguments, which are
the names of the servers that are splitting or joining, or *.net *.split
and *.net *.join if the server has chosen to hide links.

Clients that do not understand the `netsplit` and `netjoin` batch types
can safely interpret the batched QUITs and JOINs as standard QUITs
and JOINs.

## Example

An example netsplit is as follows:

    :irc.host BATCH +yXNAbvnRHTRBv netsplit irc.hub other.host
    @batch=yXNAbvnRHTRBv :aji!a@a QUIT :irc.hub other.host
    @batch=yXNAbvnRHTRBv :nenolod!a@a QUIT :irc.hub other.host
    @batch=yXNAbvnRHTRBv :jilles!a@a QUIT :irc.hub other.host
    :irc.host BATCH -yXNAbvnRHTRBv

An example netjoin is as follows:

    :irc.host BATCH +4lMeQwsaOMs6s netjoin irc.hub other.host
    @batch=4lMeQwsaOMs6s :aji!a@a JOIN #atheme
    @batch=4lMeQwsaOMs6s :nenolod!a@a JOIN #atheme
    @batch=4lMeQwsaOMs6s :jilles!a@a JOIN #atheme
    @batch=4lMeQwsaOMs6s :nenolod!a@a JOIN #ircv3
    @batch=4lMeQwsaOMs6s :jilles!a@a JOIN #ircv3
    @batch=4lMeQwsaOMs6s :Elizacat!a@a JOIN #ircv3
    :irc.host BATCH -4lMeQwsaOMs6s

## Errata

For consistency with capabilities and tags these types were renamed to lower case
(from `NETSPLIT` to `netsplit`).
