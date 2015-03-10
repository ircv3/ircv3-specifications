`REGISTER` and `VERIFY` command specification
---------------------------------------------

Copyright (c) 2014 William Pitcock <nenolod@dereferenced.org>.

Unlimited redistribution and modification of this document is allowed
provided that the above copyright notice and this permission notice
remains in tact.

## Description

The `REGISTER` and `VERIFY` commands provide a unified, standardized interface for
account creation and validation to the network's authentication layer.  This allows for a
network to provide a common interface regardless of what authentication layer they have chosen
for their network to operate, i.e. traditional services, a different central authority, or a
decentralized model similar to [IRCX][ircx].

   [ircx]: http://en.wikipedia.org/wiki/IRCX
   
We believe that the benefit of adding a common interface for account creation and validation,
is helpful to the end user in discovering the capabilities offered by the authentication and
authority layer of the network.  Further, we believe it is helpful to client authors, so that
they may provide guided account creation and validation for their users, regardless of network
configuration.

## The `REGISTER` command

The `REGISTER` command signals the intent of a client to register an account in the network's
authentication layer.  It is similar to current methods of signalling that intent.

A `REGISTER` command consists of the following format:

`REGISTER <accountname> <email> :<passphrase>`

A passphrase MAY have spaces in it.

The IRC server MAY forward the `REGISTER` command to a central authority, or process it locally.

The IRC server MUST NOT handle the `REGISTER` command as an alias.

Upon success, the IRC server MUST send the `RPL_REGISTRATIONSUCCESS` numeric, which looks like:

    :<server> 920 <user_nickname> <accountname> :Account registered
    
Upon error, the IRC server MUST send an error code that is relevant.  We suggest these numerics:

`ERR_ACCOUNT_ALREADY_EXISTS`:
    `:<server> 921 <user_nickname> <accountname> :Account already exists`
`ERR_REG_INVALID_PARAMS`:
    `:<server> 922 <user_nickname> <accountname> :Invalid params: <descriptive_text>`
    
The server MAY send additional informative text upon registration success or failure using `RPL_TEXT` or `NOTICE`.

## The `VERIFY` command

The `VERIFY` command signals the intent of a client to submit a verification token for their
account to the authentication layer.

A `VERIFY` command consists of the following format:

`VERIFY <accountname> <auth_code>`

The IRC server MAY forward the `VERIFY` command to a central authority, or process it
locally.

The IRC server MUST NOT handle the `VERIFY` command as an alias.

Upon success, the IRC server MUST send the `RPL_VERIFYSUCCESS` numeric, which looks like:

    :<server> 923 <user_nickname> <accountname> :Account verification successful

Upon error, the IRC server MUST send an error code that is relevant.  We suggest these
numerics:

`ERR_ACCOUNT_ALREADY_VERIFIED`:
    `:<server> 924 <user_nickname> <accountname> :Account already verified`
`ERR_ACCOUNT_INVALID_VERIFY_CODE`:
    `:<server> 925 <user_nickname> <accountname> :Invalid verification code`