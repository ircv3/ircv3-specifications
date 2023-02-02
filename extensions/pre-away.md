---
title: "`pre-away` Extension"
layout: spec
meta-description: A capability allowing client connections to be marked AWAY during connection registration
work-in-progress: true
copyrights:
  -
    name: "Shivaram Lingamneni"
    period: "2023"
    email: "slingamn@cs.stanford.edu"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `pre-away` CAP name. Instead, implementations SHOULD use the `draft/pre-away` CAP name to be interoperable with other software implementing a compatible work-in-progress version. The final version of the specification will use an unprefixed CAP name.

## Introduction
Some IRC server implementations offer a mode of operation where a single nickname can be associated with multiple concurrent client connections, or no client connections. Typically, such implementations are bouncers (i.e., intermediaries between the client and another server), but some are full server implementations.

Such implementations may wish to update publicly visible state depending on the status of the user's actual client connections. For example, if the user has no active connections, it may be desirable to mark them as AWAY, then mark them un-AWAY if they reconnect. However, a client implementation may wish to connect without active involvement from the user, e.g. to retrieve [chathistory][chathistory], in which case it would be undesirable to suggest that the user is present. This extension provides a mechanism for such clients to flag their connections as automatically initiated, so servers can disregard them for this or other purposes related to user presence.

## Implementation
This specification introduces a new informational capability, `draft/pre-away`. Clients MAY request this capability; servers implementing this specification MUST acknowledge it if it is requested.

Servers implementing this specification MUST accept the `AWAY` command before connection registration has completed, whether or not the `draft/pre-away` capability has been negotiated. The `AWAY` command has its normal semantics in this context, except servers SHOULD treat the form:

    AWAY *

i.e. an `AWAY` message consisting of the single character `*`, as indicating that the user is not present for an unspecified reason.

In its conventional form:

    AWAY :Gone to lunch.  Back in 5

the `AWAY` command MAY be used pre-registration to set a human-readable away message associated with the connection as usual. Similarly, `AWAY` with no parameters, which normally indicates that the user is present, has no effect.

If the client's nickname was not already present on the server, then `AWAY` pre-registration sets the away message but does not inhibit reporting of the change in nickname status, e.g. via [monitor][monitor].

Clients receiving `*` as an away message (for example, in `301 RPL_AWAY` or [away-notify][away-notify]) SHOULD treat it as indicating that the user is not present for an unspecified reason. Servers MAY substitute a human-readable message for the `*` if it would otherwise be relayed as an away message.

## Implementation considerations
This section is non-normative.

In general, the server-side aggregation of away statuses across multiple connections is outside the scope of this specification. However, in most cases, an away message of `*` should be treated as though the connection did not exist at all (for example, it should not supersede a human-readable `AWAY` message set by another connection, even if it is more recent).
