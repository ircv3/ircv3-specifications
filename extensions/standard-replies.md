---
title: Standard Replies
layout: spec
copyrights:
  -
    name: "Daniel Oaks"
    period: "2019"
    email: "daniel@danieloaks.net"
---

## Introduction

This document specifies the standard `FAIL`, `WARN`, and `NOTE` messages, intended to provide a clean, consistent interface for sending general errors, warnings, and informational messages to clients. Implementers should not need to reserve new numerics for error, warning, or general informational messages, especially as numerics themselves and the mapping of numerics to names can be unclear or conflicting.

The `FAIL` message indicates a complete failure to process a given command/function, or simply some error about the current session that clients should be aware of.

The `WARN` message indicates some non-fatal feedback about a given command/function, or some less vital feedback on the current session.

The `NOTE` message indicates some informational message about a given command/function, or about the current session.


## Message Format

The `FAIL`, `WARN`, and `NOTE` messages have the same format. Here, we describe the basic format (using `<type>` as a placeholder for the specific message type), before covering some of the differences of each message type.

    <type> <command> <code> [<context>...] :<description>

Some non-normative examples:

    FAIL * NEED_REGISTRATION :You need to be registered to continue
    NOTE AUTHENTICATE ACCOUNT_PASSPHRASE_UPDATED :Your passphrase hash has been automatically updated to use our new, more secure method

Here's what each parameter means:

- `<type>`: Either `FAIL`, `WARN`, or `NOTE`, this indicates the message type.
- `<command>`: Indicates the user command which this reply is related to, or is `*` for messages initiated outside client commands (for example, an on-connect message).
- `<code>`: Machine-readable reply code representing the meaning of the message to client software.
- `<context>`: Optional parameters that give humans extra context as to where and why the reply was spawned (for example, a particular subcommand or sub-process).
- `<description>`: A required plain-text description of the reply which is shown to users.

`<type>` is the verb of the reply, and is one of `FAIL` (indicating that a command/function could not be processed, or some significant issue with the current session), `WARN` (indicating some non-fatal feedback about a command/function or the session), or `NOTE` (indicating an informational message).

`<command>` is a case-insensitive, required parameter which is either a command name (e.g. `MODE`, `AUTHENTICATE`, `NICK`, etc), or is `*` to indicate a message that was not spawned from a particular command. This message SHOULD also be given a [label](./labeled-response.html), if one was provided by the client with the original command that spawned this reply.

`<code>` is a case-insensitive, required parameter which lets the client software handle the message appropriately. The [IRCv3 Reply Code Registry](http://ircv3.net/registry.html) lists the reply codes currently in use, and implementers MUST use an existing code if one is already defined.

Reply codes (especially more widely-applicable ones such as `NO_SUCH_NICK`) may require different action or mean different things when referring to different commands. Because of this, clients acting on an incoming reply code should also take the `<command>` into account, or ensure that the given `<code>` is intended to be globally-unique.

`<context>` is a set of optional parameters that give extra context to humans debugging and working out why a certain reply was occurred. These parameters are not intended for end users, but for developers gathering more information.

`<description>` is a required final parameter, containing a plain-text description of the reply message. For example, for a `FAIL` message, the description should clearly state what error occurred, and how the user should act to resolve it. Implementers should write their descriptions to be easily understood by end users, for example _"Passwords must contain lowercase and uppercase letters, and be 8 or more characters"_ is a more clear description than _"Password consistency and strength enumerability (sec/* internal) check failed"_.

### Capabilities

This specification adds the `standard-replies` capability. This capability allows servers to send arbitrary standard replies other than ones enabled by a specific specification. It also allows clients to indicate support for receiving any well-formed standard reply, whether or not it is recognised and used.

It is RECOMMENDED that servers use standard replies instead of vendor-specific error numerics or server notices for non-standardised features when responding to a client which supports this capability. It is NOT RECOMMENDED that servers which support this capability add new custom error numerics. Servers which support this capability SHOULD NOT replace standardised error numerics with standard replies, unless the replacement is explicitly described by some other specification.



## Examples

The below are non-normative examples showing how a server may wish to respond to various failing situations. For actual reply code usage, check the relevant specification(s).

### Account registration attempt

    Client: ACC REGISTER * mailto:dan@examplecom passphrase :this is my passphrase
    Server: FAIL ACC REG_INVALID_CALLBACK REGISTER :Email address is not valid

In this example, the command is `ACC`, with the argument `REGISTER`. The code is `REG_INVALID_CALLBACK`, and the description tells the user that the email address they've given is invalid.

### Box Stacking

    Client: BOX STACK CLOCKWISE UPRIGHT FROM_START b1,x2,z3
    Server: FAIL BOX BOXES_INVALID STACK CLOCKWISE :Given boxes are not supported

In this example, the command is `BOX`, with the arguments `STACK` and `CLOCKWISE`. The code is `BOXES_INVALID`, and the description tells the user that the boxes they've given are not supported.

### Rehash Failure

    Client: REHASH
    Server: FAIL REHASH CONFIG_BAD :Could not reload config from disk

In this example, the command is `REHASH`, with no arguments. The code is `CONFIG_BAD`, and the description tells the user that the config could not be reloaded from disk.

### Rehash Warning

    Client: REHASH
    Server: WARN REHASH CERTS_EXPIRED :Certificate [blahblah.irc.example.com] has expired

In this example, the command is `REHASH`, with no arguments. The code is `CERTS_EXPIRED`, and the description tells the user that the certificates have expired, but the rehash still completed successfully.

### Note on connecting to the network

    Server: NOTE * OPER_MESSAGE :Registering new accounts and channels has been disabled temporarily while we deal with the spam. Thanks for flying ExampleNet! -dan

In this example, this reply isn't initiated by any particular command. The code is `OPER_MESSAGE`, and the description indicates the message the oper is giving to newly-connected users.

### Message History

    Server: FAIL * ACCOUNT_REQUIRED_TO_CONNECT :An account is required to connect to the network

In this example, the failure isn't initiated by any particular command. The code is `ACCOUNT_REQUIRED_TO_CONNECT`, and the description tells the user that they need an account to connect to the network.
