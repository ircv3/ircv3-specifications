---
title: IRCv3.3 Metadata
layout: spec
copyrights:
  -
    name: "Kiyoshi Aman"
    period: "2016"
    email: "kiyoshi.aman@gmail.com"
---
## Introduction

This specification extends the [IRCv3.2 metadata specification][metadata] and
incorporates it by reference.

## METADATA

### METADATA PRIORITY

This subcommand provides a mechanism for clients to inform the server what
priorities to assign to particular keys. The default priority is 'on
subscription' for all keys. The format of `METADATA PRIORITY` MUST be as
follows:

    METADATA <Scope> PRIORITY <Priority> :<Keys>

`<Scope>` is one of `global`, `channel`, or `user`, reflecting all subscriptions,
channel-related subscriptions, and subscriptions via `MONITOR`.

`<Priority>` operates in descending order, with `sub` enabling on-subscription
and on-change notifications, `change` permitting only on-change notifications,
and `request` requiring that the client explicitly request metadata with the
relevant keys.

If any key is invalid, that key MUST generate an `ERR_KEYINVALID` event. If any
key is not intended to be visible to the user due to permissions, that key MUST
generate an `ERR_KEYNOPERMISSION` or `ERR_NOMATCHINGKEY` event. As usual, if
the target is invalid, the whole command receives `ERR_TARGETINVALID`.

## Numerics

For the purposes of reporting that a `METADATA` change was blocked rather than
silently filtered, numeric 763 is to be defined as follows:


| No. | Label                 | Format                                   |
| --- | --------------------- | ---------------------------------------- |
| 763 | `ERR_VALUEINVALID`    | `<Target> <Key> :was blocked; <Reason>`  |

`<Reason>` contains a localized reason for the value to have been rejected. 

## Examples

...

[metadata]: http://ircv3.net/specs/core/metadata-3.2.html

