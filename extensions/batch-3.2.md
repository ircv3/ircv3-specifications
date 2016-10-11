---
title: IRCv3.2 `batch` Extension
layout: spec
copyrights:
  -
    name: "William Pitcock"
    period: "2012"
    email: "nenolod@dereferenced.org"
  -
    name: "Kythyria Tieran"
    period: "2014"
    email: "kythyria@berigora.net"
  -
    name: "Alexey Sokolov"
    period: "2014"
    email: "alexey-irc@asokolov.org"
---
## The `batch` client capability

This extension describes a capability which causes a new verb to be sent to
clients when the IRC server wishes to designate that a series of events are
related to each other.

Use-cases include visual suppression of netsplits and netjoins. 

This client capability MUST be named `batch`.

## The `BATCH` verb

The IRC server will send a `BATCH` verb when it wants the client to begin
batching events together, and send another `BATCH` verb when the transaction
is concluded.

The following syntax MUST be used when starting a batch:

	BATCH +reference-tag type [<parameter 1> [<parameter 2> ... [<parameter n>]]]

The following syntax MUST be used when ending a batch:

	BATCH -reference-tag

The reference tag MUST be treated as an opaque identifier.
Reference tag MUST contain only ASCII letters, numbers, and/or hyphen, and MUST be case-sensitive.

The `type` parameter indicates how messages are associated.
The meaning of the additional parameters (if any) depend on the type.

The batched events MUST use `batch` message tag referring to the batch's reference tag.
Messages before start and after end of a batch (including start and end messages of that batch) MUST NOT refer to that batch.

For every started batch, server MUST end that batch.
Client MAY delay processing the batched events until they have received and/or processed the end of the batch.

An empty batch is allowed, and SHOULD be treated as a no-op, unless demanded otherwise
by the specification for that batch type.

Start and end messages of one batch MAY refer to another batch.
If they do, they both MUST have batch tag, and they MUST refer to the same batch.
This means that one batch is nested to another.
Client MAY delay processing messages of inner batches to the end of outermost batch.

## Specifications for batch types

Batch types are specified by IRCv3 extensions, vendor-specific types MUST be
prefixed the same way as how vendor-specific capabilities are prefixed.
See [capability negotiation](../core/capability-negotiation-3.1.html) for the
exact details.

Clients MUST NOT attempt to parse the full batch type name as anything more than an opaque identifier.

While batch types are similar to capabilities in this aspect, unlike
capabilities, batch types are not advertised by servers nor explicitly
requested by clients.

Client MAY use the type to change how the messages are presented to the user,
e.g. by collapsing them into one message.
Client MAY ignore the type and process messages one by one.
For example, that happens if the type is unknown to the client.

To submit a new batch type please follow the extension submission procedure
[found here](/participation.html).

 * [IRC Version 3.2: `netsplit` and `netjoin`](batch/netsplit-3.2.html)

## Examples

### A simple batch

	:irc.host BATCH +yXNAbvnRHTRBv netsplit irc.hub other.host
	@batch=yXNAbvnRHTRBv :aji!a@a QUIT :irc.hub other.host
	@batch=yXNAbvnRHTRBv :nenolod!a@a QUIT :irc.hub other.host
	:nick!user@host PRIVMSG #channel :This is not in batch, so processed immediately
	@batch=yXNAbvnRHTRBv :jilles!a@a QUIT :irc.hub other.host
	:irc.host BATCH -yXNAbvnRHTRBv

### Interleaving batches

	:irc.host BATCH +1 example.com/foo
	@batch=1 :nick!user@host PRIVMSG #channel :Message 1
	:irc.host BATCH +2 example.com/foo
	@batch=1 :nick!user@host PRIVMSG #channel :Message 2
	@batch=2 :nick!user@host PRIVMSG #channel :Message 4
	@batch=1 :nick!user@host PRIVMSG #channel :Message 3
	:irc.host BATCH -1
	@batch=2 :nick!user@host PRIVMSG #channel :Message 5
	:irc.host BATCH -2

Client may show the messages on the channel in order 1,2,3,4,5, or in order 1,4,2,3,5, among others.

### Nested batch

	:irc.host BATCH +outer example.com/foo
	@batch=outer :irc.host BATCH +inner example.com/bar
	@batch=inner :nick!user@host PRIVMSG #channel :Hi
	@batch=outer :irc.host BATCH -inner
	:irc.host BATCH -outer

Notice that PRIVMSG line will be processed when batch `outer` ends,
because end of batch `inner` is tagged by batch `outer`.
The order in which these two batches start doesn't matter.


## Errata

Previous versions of this spec did not specify that the full batch type name should be parsed as
nothing more than an opaque identifier. This was added to improve client resiliency.
