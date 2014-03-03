
# MONITOR - Protocol for notification of when clients become online/offline

Copyright (c) Lee Hardy <lee -at- leeh.co.uk>, Kiyoshi Aman
<kiyoshi.aman@gmail.com>.

# Introduction

Currently, ISON requests by clients use a large amount of bandwidth.  It is
expected that it is more efficient for this to be done by the server at the 
expense of cpu cycles.  The WATCH implementation that was designed to counter 
this is undocumented and apparently differs between implementations. This
protocol was designed to provide a cleaner implementation. This specification
deprecates and replaces the `WATCH` command and related functionality.

# `MONITOR` Command

The command used throughout this specification is `MONITOR`.

Each use of the `MONITOR` command takes a special modifier, indicating
the operation being performed.  The client MUST NOT attempt to specify
more than one modifier.  Only one special modifier may be used per `MONITOR`
command.

Thus it is impossible to combine additions to the list with removals from
the list -- these MUST be done with two seperate commands.

In commands and numerics where multiple targets may occur, the length of
the target list is limited only by the buffer size of 512 chars, as
defined in RFC1459.

Support of this specification is indicated by the MONITOR token in
`RPL_ISUPPORT` (005).  This token takes an optional parameter, of the maximum
amount of targets a client may have in their monitor list.  If no parameter
is specified, there is no limit.  A typical token would be:

`MONITOR=100`

For this specification, 'target' MUST be a valid nick or a valid mask as
determined by the IRC daemon.

## `MONITOR + target[,target2]*`

Adds the given list of targets to the list of targets being monitored.

If any of the targets being added are online, the server will generate
`RPL_MONONLINE` numerics listing those targets that are online.

If any of the targets being added are offline, the server will generate
`RPL_MONOFFLINE` numerics listing those targets that are offline.

## `MONITOR - target[,target2]*`

Removes the given list of targets from the list of targets being
monitored.  No output will be returned for use of this command.

## `MONITOR C`

Clears the list of targets being monitored.  No output will be returned
for use of this command.

## `MONITOR L`

Outputs the current list of targets being monitored.  All output will use
`RPL_MONLIST`, and the output will be terminated with `RPL_ENDOFMONLIST`.

## `MONITOR S`

Outputs for each target in the list being monitored, whether the client is
online or offline.  All targets that are online will be sent using 
`RPL_MONONLINE`, all targets that are offline will be sent using
`RPL_MONOFFLINE`.

# Numeric replies

## 730 - `RPL_MONONLINE`

`:<server> 730 <nick> :target[,target2]*`

This numeric is used to indicate to a client that either a target has just
become online, or that a target they have added to their monitor list is
online.

The server may send "*" instead of the target nick (`<nick>`). (This makes it
possible to send the exact same message to all clients monitoring a certain
target.)

## 731 - `RPL_MONOFFLINE`

`:<server> 731 <nick> :target[,target2]*`

This numeric is used to indicate to a client that either a target has just
left the irc network, or that a target they have added to their monitor
list is offline.

The argument is a chained list of targets that are offline.

As with 730, the server may send "*" instead of the target nick.

## 732 - `RPL_MONLIST`

`:<server> 732 <nick> :target[,target2]*`

This numeric is used to indicate to a client the list of targets they have
in their monitor list.

## 733 - `RPL_ENDOFMONLIST`

`:<server> 733 <nick> :End of MONITOR list`

This numeric is used to indicate to a client the end of a monitor list.

## 734 - `ERR_MONLISTFULL`

`:<server> 734 <nick> <limit> <targets> :Monitor list is full.`

This numeric is used to indicate to a client that their monitor list is
full, so the command failed.  The `<limit>` parameter is the maximum number of
targets a client may have in their list, the `<targets>` parameter is the
list of targets, as the client sent them, that cannot be added.


