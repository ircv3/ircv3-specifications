# IRCv3 Client Capability Negotiation

Copyright (c) 2004-2012 Kevin L. Mitchell <klmitch@mit.edu>.

Copyright (c) 2004-2012 Perry Lorier <isomer@undernet.org>.

Copyright (c) 2004-2012 Lee Hardy <lee@leeh.co.uk>.

Copyright (c) 2009-2012 William Pitcock <nenolod@atheme.org>.

Unlimited redistribution and modification of this document is allowed
provided that the above copyright notice and this permission notice
remains in tact.

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
required subcommand, optionally followed by a single parameter of space-seperated capability
identifiers.  Each capability in the list may be preceded by a capability modifier.

The subcommands for `CAP` are: `LS`, `LIST`, `REQ`, `ACK`, `NAK`, `CLEAR` and `END`.

The `LS`, `LIST`, `REQ`, `ACK` and `NAK` subcommands may be followed by a single parameter
containing a space-seperated list of capabilities.  If more than one capability is named,
the RFC1459 designated sentinel (`:`) for a multi-parameter argument must be present. The
list of capabilties MUST be parsed from left to right and capabilities SHOULD only be sent
once per command. If a capability is sent multiple times, the last one received takes priority.

If a client sends a subcommand which is not in the list above or otherwise issues an
invalid command, then numeric 410 (ERR_INVALIDCAPCMD) should be sent.  The first parameter
after the client identifier (usually nickname) should be the commandname; the second parameter
should be a human-readable description of the error.

Replies from the server must contain the client identifier name or asterisk if one is not yet
available.

The client MUST be able to use the `CAP` command anytime, even after registration.

### The CAP LS subcommand

The LS subcommand is used to list the capabilities supported by the server.  The client
should send an LS subcommand with no other arguments to solicit a list of all capabilities.

If a client issues an LS subcommand, registration must be suspended until an END subcommand
is received.

Example:

    Client: CAP LS
    Server: CAP * LS :multi-prefix sasl

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
connection.  It's sole parameter must be a list of space-separated capability identifiers.
Each capability identifier may be prefixed with a dash (`-`) to designate that the capability
should be disabled.

The capability identifier set must be accepted as a whole, or rejected entirely.

If a client issues a REQ subcommand, registration must be suspended until an END subcommand
is received.

Example:

    Client: CAP REQ :multi-prefix sasl
    Server: CAP * ACK :multi-prefix sasl

### The CAP ACK subcommand

The ACK subcommand has three uses:

* the server sends it to acknowledge a REQ subcommand;
* the server sends it to acknowledge a CLEAR subcommand;
* the client sends it to acknowledge capabilities which require client-side acknowledgement.

If an ACK reply originating from the server is spread across multiple lines, a client MUST NOT
change capabilities until the last ACK of the set is received. Equally, a server MUST NOT change
the capabilities of the client until the last ACK of the set has been sent.

In the first usage, acknowledging a REQ subcommand, the ACK subcommand has a single parameter
consisting of a space separated list of capability names, which may optionally be preceded with
one or more modifiers. 

The second usage, acknowledging a CLEAR subcommand, is similar to the first usage. When a CLEAR
subcommand is issued, all non-"sticky" capabilities are disabled, and a set of ACK subcommands
will be generated by the server with the disable modifier preceding each capability. 

The third usage is when, in the preceding two cases, some capability names have been preceded
with the ack modifier. ACK in this case is used to fully enable or disable the capability. Clients
MUST NOT issue an ACK subcommand for any capability not marked with the ack modifier in a
server-generated ACK subcommand.

### The CAP NAK subcommand

The NAK subcommand designates that the requested capability change was rejected.  The server
MUST NOT make any change to any capabilities if it replies with a NAK subcommand.

The argument of the NAK subcommand MUST consist of at least the first 100 characters of the
capability list in the REQ subcommand which triggered the NAK.

### The CAP CLEAR subcommand

The CLEAR subcommand requests that the server clear the capability set for the client.
The server MUST respond with a set of ACK subcommands indicating the capabilities being
deactivated. 

### The CAP END subcommand

The END subcommand signals to the server that capability negotiation is complete and requests
that the server continue with client registration. If the client is already registered, this
command MUST be ignored by the server.

Clients that support capabilities but do not wish to enter negotiation SHOULD send CAP END
upon connection to the server. 

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

There are two capability namespaces:

* Vendor-Specific: Names which contain a slash character (`/`) designate a vendor-specific
  capability namespace. These names are prefixed by a valid DNS domain name.
  For example: `znc.in/server-time`.  In cases if the domain name contains non-ASCII characters,
  punycode MUST be used, e.g. `xn--e1afmkfd.org/foo`.
  Vendor-Specific capabilities should be submitted to the IRCv3 working group for consideration.

* Standardized: Names for which a corresponding document sits in the IRCv3 Extension Registry.
  Names in the IRCv3 Extension Registry are reserved for your capability.

## Capability Modifiers

There are three capability modifiers specified by this standard.  If a capability modifier is
to be used, it MUST directly procede the capability identifier.

The capability modifiers are:

* `-` modifier (disable): this modifier indicates that the capability is being disabled.

* `~` modifier (ack): this modifier indicates the client must acknowledge the capability using
  an ACK subcommand.

* `=` modifier (sticky): this modifier indicates that the specified capability may not be
  disabled.
