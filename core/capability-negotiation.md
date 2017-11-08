---
title: IRCv3 Client Capability Negotiation
layout: spec
updated-by:
  - cap-notify
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
---
## What IRCv3 Client Capability Negotiation attempts to solve

IRC is an asynchronous protocol, which means that IRC clients may issue additional
IRC commands while a command is being processed.  Additionally, there is no guarantee
of a specific kind of banner being issued upon connection.  Some servers also do not
complain about unknown commands during registration, which means that a client cannot
reliably do passive implementation discovery at registration time.

If a client had to wait for a banner message, it would be incompatible with previous
versions of the IRC client protocol.

The solution to these problems is to extend the registration process with actual
capability negotiation.  If the server supports capability negotiation, the registration
process will be suspended until negotiation is completed.  If the server does not support
capability negotiation, then registration will complete immediately, and the client will
not use any IRCv3 capabilities.

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
[`cap-notify`](../extensions/cap-notify-3.2.html) capability, as described in that
specification.

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

The ACK subcommand has two uses:

* the server sends it to acknowledge a REQ subcommand;
* the client sends it to acknowledge capabilities which require client-side acknowledgement.

If an ACK reply originating from the server is spread across multiple lines, a client MUST NOT
change capabilities until the last ACK of the set is received. Equally, a server MUST NOT change
the capabilities of the client until the last ACK of the set has been sent.

In the first usage, acknowledging a REQ subcommand, the ACK subcommand has a single parameter
consisting of a space separated list of capability names, which may optionally be preceded with
one or more modifiers. 

The second usage is when, in the preceding two cases, some capability names have been preceded
with the ack modifier. ACK in this case is used to fully enable or disable the capability. Clients
MUST NOT issue an ACK subcommand for any capability not marked with the ack modifier in a
server-generated ACK subcommand.

### The CAP NAK subcommand

The NAK subcommand designates that the requested capability change was rejected.  The server
MUST NOT make any change to any capabilities if it replies with a NAK subcommand.

The argument of the NAK subcommand MUST consist of at least the first 100 characters of the
capability list in the REQ subcommand which triggered the NAK.

### The CAP END subcommand

The END subcommand signals to the server that capability negotiation is complete and requests
that the server continue with client registration. If the client is already registered, this
command MUST be ignored by the server.

Clients that support capabilities but do not wish to enter negotiation SHOULD send CAP END
upon connection to the server.

### The CAP NEW and CAP DEL subcommands

The subcommands `NEW` and `DEL` are introducted by the [`cap-notify` specification](../extensions/cap-notify-3.2.html).

## Capability Negotiation Procedure

Clients should take one of the following actions upon connection:

* issue a CAP LS subcommand with an empty capability list to discover available capabilities;
* issue a CAP REQ subcommand to request a particular set of capabilities blindly;
* issue a CAP END subcommand to immediately end capability negotiation.

While a client is permitted to not issue any CAP commands upon connection, this may have
unintentional side effects (such as forcing a downgrade to RFC1459 client protocol).

Once capability negotiation is completed with the END subcommand, registration should
continue as normal.

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
