---
title: IRCv3 WebIRC extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Evan Magaliff"
    period: "2017"
    email: "muffinmedic@kiwiirc.com"
  -
    name: "Darren Whitlen"
    period: "2017"
    email: "prawnsalad@kiwiirc.com"
---
## Notes for implementing work-in-progress version
This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `WEBIRC` command. Instead, implementations SHOULD use the `draft/WEBIRC` command to be interoperable with other software implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed command.

## Description
When a user connects through an indirect connection to the IRC server, the user's actual IP address and hostname are not visible. The `WEBIRC` command allows the intermediate server to pass on the user's actual IP address and hostname to the IRC server. This information can be utilized by the network as if the user was using a direct connection.

## Format
The `WEBIRC` command takes three parameters: `pass `, `user`, and `ip`. An optional fourth paramater, `hostname`, MAY be included. `pass` is the password authenticating the WebIRC connection to the IRC server, `user` is the user being spoofed, and `ip` is the IP address if the connecting user. If included, `hostame` is the resolved hostname of the user. If the IP cannot be resolved and no hostname is provided, the IP MUST NOT be sent in place of the `host`.

If the hostname and IP do not resolve to each other, the provided hostname SHOULD be sent and SHOULD NOT be altered (see [Security Considerations](#security-considerations)). If connection fails, the WebIRC server MUST return an `ERROR` with an appropriate message and terminate the connection.

### Examples
Hostname not included.

    WEBIRC hunter2 AzureDiamond 104.25.32.27

Hostname included.

    WEBIRC hunter2 AzureDiamond 104.25.32.27 kiwiirc.com

Error from invalid password.

    ERROR :Invalid WebIRC password

## Use Cases
Webchat applications that proxy the connection through their servers are the primary intended use case. The large amount of users connected from any single webchat gateway requires that the appropriate information be provided to the IRC server. WebIRC allows network and channel operators to utilize the full extent of moderation and secuirty tools available by exposing the user IP addresses.

Other use cases can include bouncer services that wish to pass user information to the IRC server.

## Limitations
WebIRC requires that the both WebIRC server and IRC server be configured to accept the connection, included a set password and, ideally, specified, hostnames as specified in the below security section.

## Security Considerations
WebIRC allows anyone with a valid password to successfully connect to the IRC server and spoof any desired hostname. IRC servers SHOULD use secure passwords, SHOULD limit the IP addresses of accepted WebIRC servers, and SHOULD reject any connections that do come from a valid IP using a correct password. Because the possibility for hostname spoofing exists, IRC servers SHOULD attempt to validate hostnames and match them to the IP addess. It is the responsibility of the IRC server to verify the authenticity of connecting users. To assist network operators and prevent abuse, IRC servers SHOULD show when a WebIRC connection is in use and SHOULD provide the original host when possible.

## Current Implementations
Although many IRCDs and webchat applications having current WEBIRC implementations, none currently follow this specification.