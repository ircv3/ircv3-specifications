---
title: Monitor
layout: spec
redirect_from:
  - /specs/core/monitor-3.2.html
copyrights:
  -
    name: "Lee Hardy"
    period: "2004-2015"
    email: "lee@leeh.co.uk"
  -
    name: "Kiyoshi Aman"
    period: "2013-2015"
    email: "kiyoshi.aman@gmail.com"
  -
    name: "William Pitcock"
    period: "2015"
    email: "nenolod@dereferenced.org"
---

A protocol for notification of when clients become online/offline

## Introduction

Currently, `ISON` requests by clients use a large amount of bandwidth.  It is
expected that it is more efficient for this to be done by the server at the 
expense of cpu cycles.  This specification deprecates both the `ISON` and
legacy `WATCH` extensions.

### `WATCH` vs. `MONITOR`

The `WATCH` implementation suffers from quite a few problems. First, the
implementation of the `WATCH` command is non-standard, and differs between
different vendor implementations of the `WATCH` command.

The `MONITOR` extension enhances the legacy `WATCH` command by providing
documented, standardized, `ISON` style notifications instead of one numeric
per watch-list entry as with `WATCH`. Further, the `MONITOR` implementation
is allowed to multicast notifications to every client which has a subscription
to a target whom is subject to a state change. The `MONITOR` implementation
also enhances user privacy by disallowing subscription to hostmasks,
allowing users to avoid nick-change stalking.

## `MONITOR` Command

The command used throughout this specification is `MONITOR`.

Each use of the `MONITOR` command takes a special modifier, indicating
the operation being performed.  The client MUST NOT attempt to specify
more than one modifier.  Only one special modifier may be used per `MONITOR`
command.

Thus it is impossible to combine additions to the list with removals from
the list -- these MUST be done with two separate commands.

In commands and numerics where multiple targets may occur, the length of
the target list is limited only by the buffer size of 512 chars, as
defined in RFC1459.

Support of this specification is indicated by the MONITOR token in
`RPL_ISUPPORT` (005).  This token takes an optional parameter, of the maximum
amount of targets a client may have in their monitor list.  If no parameter
is specified, there is no limit.  A typical token would be:

    MONITOR=100

For this specification, 'target' MUST be a valid nick as determined by
the IRC daemon.

### `MONITOR + target[,target2]*`

Adds the given list of targets to the list of targets being monitored.
Targets already in the list MUST NOT be added again.

If any of the targets being added are online, the server will generate
`RPL_MONONLINE` numerics listing those targets that are online.

If any of the targets being added are offline, the server will generate
`RPL_MONOFFLINE` numerics listing those targets that are offline.

### `MONITOR - target[,target2]*`

Removes the given list of targets from the list of targets being
monitored.  No output will be returned for use of this command.

### `MONITOR C`

Clears the list of targets being monitored.  No output will be returned
for use of this command.

### `MONITOR L`

Outputs the current list of targets being monitored.  All output will use
`RPL_MONLIST`, and the output will be terminated with `RPL_ENDOFMONLIST`.

### `MONITOR S`

Outputs for each target in the list being monitored, whether the client is
online or offline.  All targets that are online will be sent using 
`RPL_MONONLINE`, all targets that are offline will be sent using
`RPL_MONOFFLINE`.

## Numeric replies

### 730 - `RPL_MONONLINE`

    :<server> 730 <nick> :target[!user@host][,target[!user@host]]*

This numeric is used to indicate to a client that either a target has just
become online, or that a target they have added to their monitor list is
online.

The server MAY send a hostmask with the target.

The server may send "*" instead of the nick (`<nick>`). (This makes it
possible to send the exact same message to all clients monitoring a certain
target.)

### 731 - `RPL_MONOFFLINE`

    :<server> 731 <nick> :target[,target2]*

This numeric is used to indicate to a client that either a target has just
left the irc network, or that a target they have added to their monitor
list is offline.

The argument is a chained list of targets that are offline.

As with 730, the server may send "*" instead of the nick.

### 732 - `RPL_MONLIST`

    :<server> 732 <nick> :target[,target2]*

This numeric is used to indicate to a client the list of targets they have
in their monitor list.

### 733 - `RPL_ENDOFMONLIST`

    :<server> 733 <nick> :End of MONITOR list

This numeric is used to indicate to a client the end of a monitor list.

### 734 - `ERR_MONLISTFULL`

    :<server> 734 <nick> <limit> <targets> :Monitor list is full.

This numeric is used to indicate to a client that their monitor list is
full, so the command failed.  The `<limit>` parameter is the maximum number of
targets a client may have in their list, the `<targets>` parameter is the
list of targets, as the client sent them, that cannot be added.

## Errata

* Earlier version of this spec had RPL_MONONLINE only sending nicknames,
  This did not match existing implementations.
