userhost-in-names Client Capability Extension
---------------------------------------------

The `userhost-in-names` capability extends the NAMES reply messages
(RPL\_NAMREPLY) to contain the full hostmask (nick!user@host) of every user
listed.

Example:

    --> NAMES #atheme
    :irc.bnc.im 353 guest = #atheme :Rylee!rylai@localhost somasonic!andrew@somasonic.org
    :irc.bnc.im 366 guest #atheme :End of /NAMES list.

Historical note: Some older (pre-IRCv3) software implements the same extension
as `PROTOCTL UHNAMES` and advertises it using the `UHNAMES` token in
RPL\_ISUPPORT; some implementations support enabling the same capability using
either CAP or PROTOCTL. New servers or clients SHOULD NOT implement the
PROTOCTL method, although they may still do so for compatibility with older
software.
