# identify-msg Client Capability Extension

The `identify-msg` capability causes the server to send an "identification"
prefix in the message parameter of PRIVMSG and NOTICES commands.

The prefix is a single character, `+` (ASCII plus) if the sender is
"identified" according to services, `-` (ASCII minus) otherwise.

The definition of 'identified' varies between services software. For example:

  - "has the +r usermode set" – if services dynamically set/unset the umode;
  - "is identified to an account which owns the current nickname" – if services
    provide such information in other ways;
  - "is identified to any account" – if services don't implement nickname
    ownership at all.

For this reason, the extension must not be used for security purposes.

Example (demonstrated using the `account-notify` capability):

    :user ACCOUNT *
    :user PRIVMSG #atheme :-Hello everyone.
    :user ACCOUNT user
    :user PRIVMSG #atheme :+Now I'm logged in.
    :user NICK loser
    :loser PRIVMSG #atheme :-I'm still logged in, but my account does not match.

Historical note: There exist pre-IRCv3 servers and clients that implement the
same extension as `CAPAB IDENTIFY-MSG`. New servers and clients SHOULD NOT
implement the CAPAB command.

In fact, this extension is entirely superseded by the newer capabilities
`account-msg`, `account-notify`, and `extended-join`, which also allow for
services implementations that have account names different from nicknames. The
`identify-msg` extension therefore should not be implemented.
