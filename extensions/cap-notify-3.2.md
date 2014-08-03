cap-notify client capability specification
------------------------------------------

Copyright (c) 2014 Attila Molnar <attilamolnar@hush.com>

Unlimited redistribution and modification of this document is allowed
provided that the above copyright notice and this permission notice
remains in tact.

## Description

This client capability MUST be named `cap-notify`.

The `cap-notify` client capability allows a client to be notified when
the client capability offering (i.e. capabilities listed in `CAP LS`)
on a server changes.

When enabled, the server MUST notify clients about all new capabilities
and about existing capabilities that are no longer available via the messages
specified in this document.

The `cap-notify` capability MUST be implicitly enabled if the client requests
`CAP LS 302`, as described in the [capability negotiation 3.2 specification](/specification/capability-negotiation-3.2.md).
In that case `cap-notify` MAY be presented in response to `CAP LS`, but the
server MAY choose not to. This is to ease the adaptation of new features.

## Subcommands

This specification introduces two new CAP subcommands: `NEW` and `DEL`.

Clients implementing this specification SHOULD react to the `CAP NEW` message
with a `CAP REQ` message if they would like to use one of the newly offered
capabilities.

Upon receiving a `CAP DEL` message, the client MUST treat the listed
capabilities cancelled and no longer available.
Clients SHOULD NOT send `CAP REQ` messages to cancel the capabilities in
`CAP DEL`, as they have been canceled already by the server.

Servers MUST cancel any capability-specific behavior for a client after
sending the `CAP DEL` message to the client.

Servers SHOULD NOT remove support for in-use sticky capabilities.

Clients MUST gracefully handle situations when the server removes support
for any capability, including sticky capabilities. If the client cannot
continue to operate without a capability, disconnecting with an appropriate
QUIT message is acceptable.

The format of the new messages is as follows:

    CAP NEW :<extension 1> [<extension 2> ... [<extension n>]]
    CAP DEL :<extension 1> [<extension 2> ... [<extension n>]]

The last parameter in both new messages contain a list of new
extensions that became available on the server (in the case of `CAP NEW`)
or a list of extensions that are no longer available (for `CAP DEL`).
The capability list is space separated.

## Examples

1. Message indicating that a new extension named `batch` is now available:

	    :irc.example.com CAP modernclient NEW :batch

2. Message indicating that multiple extensions are no longer available:

	    :irc.example.com CAP modernclient DEL :userhost-in-names multi-prefix away-notify

3. Example showing a client requesting capabilities when they become available:

	    Server: :irc.example.com CAP tester NEW :away-notify extended-join
	    Client: CAP REQ :extended-join away-notify
	    Server: :irc.example.com CAP tester ACK :extended-join away-notify
