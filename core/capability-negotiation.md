---
title: IRCv3 Client Capability Negotiation
layout: spec
redirect_from:
  - /specs/core/capability-negotiation-3.1.html
  - /specs/core/capability-negotiation-3.2.html
  - /specs/extensions/cap-notify-3.2.html
copyrights:
  -
    name: "Kevin L. Mitchell"
    period: "2004-2012"
    email: "klmitch@mit.edu"
  -
    name: "Perry Lorier"
    period: "2004-2012"
    email: "isomer@undernet.org"
  -
    name: "Lee Hardy"
    period: "2004-2012"
    email: "lee@leeh.co.uk"
  -
    name: "William Pitcock"
    period: "2009-2012"
    email: "nenolod@dereferenced.org"
  -
    name: "Attila Molnar"
    period: "2014"
    email: "attilamolnar@hush.com"
  -
    name: "Daniel Oakley"
    period: "2017"
    email: "daniel@danieloaks.net"
---
## What Client Capability Negotiation attempts to solve

The IRC protocol is old, and changing it is difficult because you need to convince other
(sometimes years-old) software to support your changes. Client Capability Negotiation
allows clients and servers to negotiate new features in a backwards-compatible way – even
features that change how the protocol works in deep or extensive ways.

Client Capability Negotiation means that client and server authors can develop new
extensions to the protocol, and software can use (or not use them) as they wish.

## Connection Registration

Upon connecting to the IRC server, clients need a way to negotiate capabilities (protocol
extensions) with the server. Negotiation is done with the `CAP` command, using the
subcommands below to list and request capabilities.

Upon connecting to the server, the client attempts to start the capability negotiation
process, as well as sending the standard `NICK`/`USER` commands (to complete registration
if the server doesn't support capability negotiation).

Upon connecting to the IRC server, clients SHOULD send one of the following messages:

- `CAP LS [version]` to discover the available capabilities on the server.
- `CAP REQ` to blindly request a particular set of capabilities.

Following this, the client MUST send the standard `NICK` and `USER` IRC commands.

Upon receiving either a `CAP LS` or `CAP REQ` command during connection registration, the
server MUST not complete registration until the client sends a `CAP END` command to
indicate that capability negotiation has ended. This allows clients to request their
desired capabilities before completing registration.

Once capability negotiation has completed with a client-send `CAP END` command, registration
continues as normal.

With the above registration process, clients will either recieve a `CAP` message indicating
that registration is paused while capability negotiation happens, or registration will
complete immediately, indicating that the server doesn't support capability negotiation.

Example with capability negotiation:

    Client: CAP LS 302
    Client: NICK dan
    Client: USER d * 0 :This is a really good name
    Server: CAP * LS :multi-prefix sasl
    Client: CAP REQ :multi-prefix
    Server: CAP * ACK multi-prefix
    Client: CAP END
    Server: 001 dan :Welcome to the Internet Relay Network dan
    ...

Example without capability negotiation:

    Client: CAP LS 302
    Client: NICK dan
    Client: USER d * 0 :This is a really good name
    Server: 001 dan :Welcome to the Internet Relay Network dan
    ...

## The CAP command

The client capability negotiation extension is implemented by the addition of one command
with several subcommands.  The command added is named `CAP`.  `CAP` takes a single
required subcommand, optionally followed by a single parameter of space-separated capability
identifiers.  Each capability in the list may be preceded by a capability modifier.

The subcommands for `CAP` are: `LS`, `LIST`, `REQ`, `ACK`, `NAK`, and `END`.

The `LS`, `LIST`, `REQ`, `ACK` and `NAK` subcommands may be followed by a single parameter
containing a space-separated list of capabilities.  If more than one capability is named,
the RFC1459 designated sentinel (`:`) for a multi-parameter argument must be present. The
list of capabilities MUST be parsed from left to right and capabilities SHOULD only be sent
once per command. If a capability is sent multiple times, the last one received takes priority.

If a client sends a subcommand which is not in the list above or otherwise issues an
invalid command, then numeric 410 (ERR_INVALIDCAPCMD) should be sent.  The first parameter
after the client identifier (usually nickname) should be the commandname; the second parameter
should be a human-readable description of the error.

Replies from the server must contain the client identifier name or asterisk if one is not yet
available.

Example:

    Client: CAP FOO
    Server: :example.org 410 * FOO :Invalid CAP command

The client MUST be able to use the `CAP` command anytime, even after registration.

### The CAP LS subcommand

The LS subcommand is used to list the capabilities supported by the server.  The client
should send an LS subcommand with no other arguments to solicit a list of all capabilities.

If a client issues an LS subcommand, registration must be suspended until an END subcommand
is received.  If no capabilities are available, an empty parameter must be sent.

Example:

    Client: CAP LS
    Server: CAP * LS :multi-prefix sasl

#### CAP LS Version

The LS subcommand has an additional argument which is the version number of the latest
capability negotiation protocol supported by the client.

Clients that send `302` as the CAP LS version are presumed to support `CAP LS 302` features
for the future life of the connection.

Clients that do not send any version number with CAP LS are presumed to not support these
extra features.

If a client has not indicated support for `CAP LS 302` features, the server MUST NOT send
these new features to the client such as cap values and multiline replies.

When CAP Version `302` is enabled, the client also implicitly indicates support for the
`cap-notify` capability listed below, and support for the relevant `NEW` and `DEL`
subcommands.

Example:

    Client: CAP LS 302
    Server: CAP * LS :multi-prefix

#### Capability Values

If the client supports CAP Version `302`, the server MAY specify additional data for each
capability using the `<name>=<value>` format in `CAP LS` and `CAP NEW` replies.

Each capability, if it supports a value, defines what this value means in its' specification.

Example:

    Client: CAP LS 302
    Server: CAP * LS :multi-prefix sasl=PLAIN,EXTERNAL server-time draft/packing=EX1,EX2

#### Multiline replies to `CAP LS` and `CAP LIST`

Servers MAY send multiple lines in response to `CAP LS`. If the reply contains multiple
lines (due to IRC line length limitations), and the client supports CAP Version `302`,
all but the last reply MUST have a parameter containing only an asterisk (`*`) preceding
the capability list. This lets clients know that more CAP lines are incoming, so that it
can delay capability negotiation until it has seen all available server capabilities.

If a multi-line response to `CAP LIST` is sent, and the client supports CAP Version `302`,
the server MUST delimit all replies except for the last one sent as noted above.

Example with a client that **does not** support CAP version `302`:

    Client: CAP LS
    Server: CAP * LS :multi-prefix extended-join account-notify batch invite-notify tls
    Server: CAP * LS :cap-notify server-time example.org/dummy-cap=dummyvalue example.org/second-dummy-cap
    Server: CAP * LS :userhost-in-names sasl=EXTERNAL,DH-AES,DH-BLOWFISH,ECDSA-NIST256P-CHALLENGE,PLAIN

Example with a client that supports CAP version `302`:

    Client: CAP LS 302
    Server: CAP * LS * :multi-prefix extended-join account-notify batch invite-notify tls
    Server: CAP * LS * :cap-notify server-time example.org/dummy-cap=dummyvalue example.org/second-dummy-cap
    Server: CAP * LS :userhost-in-names sasl=EXTERNAL,DH-AES,DH-BLOWFISH,ECDSA-NIST256P-CHALLENGE,PLAIN

Example of a `LIST` reply with a client that supports CAP version `302`:

    Client: CAP LIST
    Server: CAP modernclient LIST * :example.org/example-cap example.org/second-example-cap account-notify
    Server: CAP modernclient LIST :invite-notify batch example.org/third-example-cap

### The CAP LIST subcommand

The LIST subcommand is used to list the capabilities associated with the active connection.
The client should send a LIST subcommand with no other arguments to solicit a list of
active capabilities.

If no capabilities are active, an empty parameter must be sent.

Example:

    Client: CAP LIST
    Server: CAP * LIST :multi-prefix

### The CAP REQ subcommand

The REQ subcommand is used to request a change in capabilities associated with the active
connection.  Its sole parameter must be a list of space-separated capability identifiers.
Each capability identifier may be prefixed with a dash (`-`) to designate that the capability
should be disabled.

If a client requests a capability which is already enabled, or tries to disable a capability
which is not enabled, the server MUST continue processing the REQ subcommand as though
handling this capability was successful.

The capability identifier set must be accepted as a whole, or rejected entirely.

If a client issues a REQ subcommand, registration must be suspended until an END subcommand
is received.

Example:

    Client: CAP REQ :multi-prefix sasl
    Server: CAP * ACK :multi-prefix sasl

### The CAP ACK subcommand

The ACK subcommand is sent by the server to acknowledge a client-sent REQ, and let the client
know that their requested capabilities have been enabled.

This subcommand has a single parameter consisting of a space-separated list of capability
names. Each capability name may be prefixed with a dash (`-`), indicating that this
capability has been disabled as requested.

### The CAP NAK subcommand

The NAK subcommand designates that the requested capability change was rejected.  The server
MUST NOT make any change to any capabilities if it replies with a NAK subcommand.

The argument of the NAK subcommand MUST consist of at least the first 100 characters of the
capability list in the REQ subcommand which triggered the NAK.

Example:

    Client: CAP REQ :multi-prefix sasl ex3
    Server: CAP * NAK :multi-prefix sasl ex3

### The CAP END subcommand

The END subcommand signals to the server that capability negotiation is complete and requests
that the server continue with client registration. If the client is already registered, this
command MUST be ignored by the server.

Clients that support capabilities but do not wish to enter negotiation SHOULD send CAP END
upon connection to the server.

### The CAP NEW subcommand

The NEW subcommand MUST ONLY be sent to clients that have negotiated CAP Version `302` or
enabled the `cap-notify` capability.

The NEW subcommand signals that the server supports one or more new capabilities, and may be
sent at any time. Clients that support `CAP NEW` messages should respond with a `CAP REQ`
message if they wish to enable one of the newly-offered capabilities.

The format of a `CAP NEW` message is:

    CAP <nick> NEW :<extension 1> [<extension 2> ... [<extension n>]]

As with `LS`, the last parameter is a space-separated list of new capabilities that are now
offered by the server. If the client supports CAP Version `302`, the capabilities SHOULD be
listed with values, as they would be if the client sent `CAP LS 302`.

Example:

    Server: :irc.example.com CAP modernclient NEW :batch

Example with following `REQ`:

    Server: :irc.example.com CAP tester NEW :away-notify extended-join
    Client: CAP REQ :extended-join away-notify
    Server: :irc.example.com CAP tester ACK :extended-join away-notify

### The CAP DEL subcommand

The NEW subcommand MUST ONLY be sent to clients that have negotiated CAP Version `302` or
enabled the `cap-notify` capability.

The DEL subcommand signals that the server no longer supports one or more capabilities that
have been advertised. Upon receiving a `CAP DEL` message, the client MUST treat the listed
capabilities cancelled and no longer available. Clients SHOULD NOT send `CAP REQ` messages
to cancel the capabilities in `CAP DEL`, as they have already been canceled by the server.

Servers MUST cancel any capability-specific behavior for a client after sending the
`CAP DEL` message to the client.

Clients MUST gracefully handle situations when the server removes support for any
capability. If the client cannot continue to operate without a capability, disconnecting
with an appropriate `QUIT` message is acceptable.

The format of a `CAP DEL` message is:

    CAP <nick> DEL :<extension 1> [<extension 2> ... [<extension n>]]

The last parameter is a space-separated list of capabilities that are no longer available.

Example:

    Server: :irc.example.com CAP modernclient DEL :userhost-in-names multi-prefix away-notify

Example showing new SASL mechanisms becoming available. This example requires the client to
support both `CAP LS 302` and `batch`:

        Client: CAP LS 302
        Server: :irc.example.com CAP * LS :sasl=PLAIN batch
        Client: CAP REQ batch
        ...
        Server: :irc.example.com BATCH +cap example.com/cap-value
        Server: @batch=cap :irc.example.com CAP client DEL :sasl
        Server: @batch=cap :irc.example.com CAP client NEW :sasl=PLAIN,EXTERNAL
        Server: :irc.example.com BATCH -cap

The `cap` batch indicates that (as in this example), the given `NEW`/`DEL` messages may be
displayed together for simplicity's sake.

## `cap-notify`

The `cap-notify` capability indicates support for the `NEW` and `DEL` messages listed above.
This capabilitiy MUST be implicitly enabled if the client requests `CAP LS` with a version
of 302 or newer (`CAP LS 302`), as described above.

Further, the `cap-notify` capability MAY NOT be disabled if the client requests `CAP LS`
with a version of 302 or newer.

When implicitly enabled via this mechanism, servers MAY list the `cap-notify` capability
in `CAP LS` and `CAP LIST` responses. Additionally client MAY request the capability with
`CAP REQ`, and capable servers MUST accept and `CAP ACK` the request without side effects.
This lets clients blindly enable this capability, regardless of it being implicitly
enabled by the server.

Clients that do not support CAP Version 302 MAY request the `cap-notify` capability
explicitly. Such clients MAY disable the capability at any time.  This to ease the
adaptation of new features.

When enabled, the server MUST notify the client about all new capabilities and about
existing capabilities that are no longer available using the `NEW` and `DEL` subcommands
listed above.

## Rules for naming capabilities

The full capability name MUST be treated as an opaque identifier.

There are two capability namespaces:

### Vendor-Specific

Names which contain a slash character (`/`) designate a vendor-specific capability namespace.

These names are prefixed by a valid DNS domain name.

For example: `znc.in/server-time`.

In cases where the prefix contains non-ASCII characters, punycode MUST be used,
e.g. `xn--e1afmkfd.org/foo`.

Vendor-Specific capabilities should be submitted to the IRCv3 working group for consideration.

### Standardized

Names that are on the [IRCv3 Capability Registry](http://ircv3.net/registry.html#capabilities) and
for which a corresponding document sits in the IRCv3 Extension Registry.

Names in the IRCv3 Capability Registry are reserved for your capability.

The IRCv3 Working Group reserves the right to reuse names which have not been submitted to the
registry. If you do not wish to submit your capability then you MUST use a vendor-specific name
(see above).

### Capability Modifiers

Prior versions of capability negotiation included 'capability modifiers', which were characters
prepended to capability names to indicate that they could not be disabled or required extra client
acknowledgement to be enabled.

These modifiers have been deprecated. Capabilities MUST NOT require an `ACK` from the client to be
activated. Instead of indicating a capability as 'sticky', servers should instead send a `NAK` to
any request to disable these capabilities.

## Errata

Previous versions of this specification referred to a CAP CLEAR command, which has been removed
because it is not useful.  We do not recommend implementing or supporting CAP CLEAR.  See
[issue #134](https://github.com/ircv3/ircv3-specifications/issues/134) for more information, including
rationale for this clarification.

Previous versions of this spec did not include specific references to re-enabling or re-disabling a
capability in the CAP REQ subcommand section. This was clarified in later versions of the
specification.

Previous versions of this spec did not specify how to handle CAP LS when a server did not support
any capabilities.  This was clarified to match CAP LIST, requiring a reply with an empty parameter.

Previous versions of this spec did not specify that the full capability name MUST be treated as
an opaque identifier. This was added to better suit real-world usage and to improve client
resiliency.

Previous versions of this spec defined 'capability modifiers'. These are not in use by software and
have been removed from this specification.

Previous versions of this spec did not specify when the `<name>=<value>` format could be used. This
was clarified to limit capability values to the `LS` and `NEW` subcommands.

Previous versions of this spec did not link to the `cap-notify` specification (nor note the fact
that it is automatically enabled for clients that enable CAP Version 302). The specification now
links to the `cap-notify` spec and lets authors know about the implicit enabling of this capability.

Previous versions of this spec mistakenly missed `<nick>` between `CAP` and `NEW`/`DEL` subcommands,
but had it in the examples anyway.

Previous versions of this spec did not mention whether `NEW` and `DEL` can have values or not.

Previous versions of this spec were missing clarification of client and server behaviour when the
capability is implicitly enabled with CAP Version `302` or newer.

Previous versions of this spec listed that `CAP ACK` could be sent from the client to server, for
capabilities that required extra client acknowledgement. This was removed since cap modifiers have
been deprecated and removed (except for `-`, which has been specified in other ways).

Previous versions of this spec listed connection registration in the middle. This section has been
rewritten and placed near the front, now including examples of negotiating capabilities during
initial registration.
