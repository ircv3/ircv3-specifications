---
title: IRCv3 nicklist extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Evan Magaliff"
    period: "2017"
    email: "evan@muffinmedic.net"
---
## Description
This document describes the format of the `nicklist` capability extension. 

This specification allows for sending a list of nicks the client would like to use to the server. The server then attempts to change the user's nick to each in the list until a successful nick change occurs or the list is exhausted.

The given nicklist takes the format: `:foo!bar@example.com NICK :nick1 nick2 nick3`

If a nick is available and valid, the server should return a successful nick change and further processing of the given list halted. If a nickname is invalid, the server should return a `432` and proceed to attempt the next nick. If a nickname is already in use, a `433` should be returned and the next nick attempted. The server should return a `431` if no nicks are given.

## Use cases
This specification is useful for instances where a user's primary nick is not available and forced obtainment of such is not available, either because services are not supported or are not currently available (netsplit, services outage). A user would be able to provide the server a list of nicks it would like and have the first available one used. This method is more efficient than sending a nick, waiting for a reply, and attempting the next one.

## Limitations
A method does not currently exist to inform the client that all given nicks cannot be used for the above reasons. A numeric indicating NICKSNOTAVAILABLE is required, or the client must match the number of replies to the number of sent nicks and inform the user a nick change could not occur.

Only one attempt by the server is given to each nick. If no nicks are available (in use or erroneous), the client must send a new `NICK` command to the server.