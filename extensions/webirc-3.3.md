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
## Purpose
This specification is intended to document the existing `WEBIRC` command that is already in use by multiple IRCDs and WebIRC gateway to provide a standard specification for implementation. This specification does not add any new features or changes, although new features, such as use of public-key cryptography over password authentication, can be submitted as extensions.

## Description
When a user connects through an indirect connection to the IRC server, the user's actual IP address and hostname are not visible. The `WEBIRC` command allows the WebIRC gateway to pass on the user's actual IP address and hostname to the IRC server. This information can be utilized by the network as if the user was connecting directly.

## Format
The `WEBIRC` command takes four parameters: `pass `, `user`, `hostname`, and `ip`. `pass` is the password authenticating the WebIRC connection to the IRC server which must be agreed upon ahead of time, `user` is the user being spoofed or WebIRC gateway service name, `hostname` is the resolved hostname of the user's IP address, and `ip` is the IPv4 or IPv6 address of the connecting user (IPv4-in-IPv6 MUST NOT be sent). If the IP address cannot be resolved, the IP address MUST be sent as both the `hostname` and `ip`.

If forward and reverse DNS do not match, the IP addesss SHOULD be sent as the `hostname` and the unverified `hostname` SHOULD NOT be sent (see [Security Considerations](#security-considerations)). If connection fails, the WebIRC gateway MUST return an `ERROR` with an appropriate message and terminate the connection.

### Examples
Generic format.

    WEBIRC password user hostname ip

IP address resolves to hostname.

    WEBIRC hunter2 AzureDiamond kiwiirc.com 104.25.32.27

IP address does not resolve to hostname.

    WEBIRC hunter2 AzureDiamond 104.25.32.27 104.25.32.27

Error from invalid password.

    ERROR :Invalid WebIRC password

## Use Cases
Webchat applications that proxy the connection through their servers are the primary intended use case. The large amount of users connected from any single webchat gateway requires that the appropriate information be provided to the IRC server. WebIRC allows network and channel operators to utilize the full extent of moderation and secuirty tools available by exposing the user IP addresses.

Other use cases can include bouncer services that wish to pass user information to the IRC server.

## Limitations
WebIRC requires that the both WebIRC gateway and IRC server be configured to accept the connection in advance. Specific reccomendations are outlined below in the [Security Considerations](#security-considerations) section.

## Security Considerations
WebIRC allows anyone with a valid password to successfully connect to the IRC server and spoof any desired hostname. IRC servers SHOULD use secure passwords, SHOULD whitelist the IP addresses for the allowed WebIRC gateway servers, and SHOULD reject any connections that attempt to use the WEBIRC command that do not come from a whitelisted IP address or using an incorrect password. Because the possibility for hostname spoofing exists, IRC servers MAY attempt to further validate or resolve hostnames or and match them to the IP addess. It is the responsibility of the IRC server to verify the authenticity of connecting users and perform additional security checks as they see fit. To assist network operators and prevent abuse, IRC servers SHOULD show when a WebIRC connection is in use and SHOULD provide the original host when possible.

## Current Implementations
Widespread adoption of WebIRC already exists for both clients and IRCDs. Below is a non-exhaustive list of current implementions to demonstrate in-use instances.
 - Clients
  - [KiwiIRC](https://kiwiirc.com/docs/webirc)
  - [Mibbit](https://wiki.mibbit.com/index.php/Webirc)
  - [qwebirc](http://qwebirc.org/features)
  - [The Lounge](https://thelounge.github.io/docs/server/configuration.html)
 - Servers
  - [charybdis](https://github.com/charybdis-ircd/charybdis/blob/release/4/extensions/m_webirc.c)
  - [Elemental IRCd](https://github.com/Elemental-IRCd/elemental-ircd/blob/master/doc/example.conf#L221)
  - [InspIRCd](https://wiki.inspircd.org/Modules/2.0/cgiirc)
  - [UnrealIRCd](https://www.unrealircd.org/docs/WebIRC_Support)