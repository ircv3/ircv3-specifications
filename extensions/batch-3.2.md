# IRC `BATCH` extension

Copyright (c) 2012 William Pitcock <nenolod@dereferenced.org>.
Copyright (c) 2014 Kythyria Tieran <kythyria@berigora.net>
Copyright (c) 2014 Alexey Sokolov <alexey-irc@asokolov.org>

Unlimited redistribution and modification is allowed provided that the above
copyright notice and this permission notice remains intact.

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

The batched events MUST use `batch` message tag refering to the batch's reference tag.

For every started batch, server MUST end that batch.
Client MAY start processing the batched events only when they have received the end of the batch.

Start of a batch MAY be inside another batch itself, but in that case messages tagged
with this new batch MUST NOT appear until this "start" message is actually processed
(until the batch containing this "start" message ends itself).

End of a batch MAY be inside another batch itself. That's useful to mark a batch as nested to the other batch.

An empty batch is allowed, and SHOULD be treated as a no-op, unless demanded otherwise
by the specification for that batch type.

## Specifications for batch types

Batch types are specified by IRCv3 extensions, vendor-specific types MUST be
prefixed the same way as how vendor-specific capabilities are prefixed.
See [capability negotiation](/specification/capability-negotiation-3.1) for the
exact details.

While batch types are similiar to capabilities in this aspect, unlike
capabilities, batch types are not advertised by servers nor explicitly
requested by clients.

Client MAY use the type to change how the messages are presented to the user,
e.g. by collapsing them into one message.
Client MAY ignore the type and process messages one by one.
For example, that happens if the type is unknown to the client.

To submit a new batch type please follow the extension submission procedure
[found here](/index).

 * [IRC Version 3.2: `NETSPLIT` and `NETJOIN`](/extensions/batch/netsplit)

## Examples

### A simple batch

	:irc.host BATCH +yXNAbvnRHTRBv NETSPLIT irc.hub other.host
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

Client should show the messages on the channel in order 1,2,3,4,5, or in order 1,2,4,3,5.

### Nested batch

	:irc.host BATCH +outer example.com/foo
	:irc.host BATCH +inner example.com/bar
	@batch=inner :nick!user@host PRIVMSG #channel :Hi
	@batch=outer :irc.host BATCH -inner
	:irc.host BATCH -outer

Notice that PRIVMSG line will be processed when batch `outer` ends,
because end of batch `inner` is tagged by batch `outer`.
The order in which these two batches start doesn't matter.

