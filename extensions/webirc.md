---
title: WebIRC Extension
layout: spec
copyrights:
  -
    name: "MuffinMedic"
    period: "2017"
  -
    name: "Darren Whitlen"
    period: "2017"
    email: "prawnsalad@kiwiirc.com"
---

## Motivation

This specification is intended to document the existing `WEBIRC` command that is already in use by multiple IRC servers and WebIRC gateways to provide a standard specification for implementation. This specification does not add any new features or changes, although new features, such as use of public-key cryptography instead of password authentication, can be submitted as extensions.

## Introduction

When a user connects through an indirect connection to the IRC server, the user's actual IP address and hostname are not visible. The `WEBIRC` command allows the WebIRC gateway to pass on the user's actual IP address and hostname to the IRC server. This information can be used by the network as if the user was connecting directly.

## Format

The `WEBIRC` command MUST be the first command sent from the WebIRC gateway to the IRC server and MUST be sent before capability negotiation.

The `WEBIRC` command takes four or five parameters: `password `, `gateway`, `hostname`, `ip`, and `options`.
- `password` Password authenticating the WebIRC gateway to the IRC server which must be agreed upon ahead of time
- `gateway` WebIRC gateway service name
- `hostname` User's resolved hostname
- `ip` IPv4 or IPv6 address of the connecting user. If the IP address cannot be resolved to a DNS name, the IP address MUST be sent as both the `hostname` and `ip`. If the IP address is an IPv6 address beginning with a colon, it MUST be sent with a canonically-acceptable preceding zero.
- `options` is an optional set of space-separated arguments detailing other information about the connection.

If forward and reverse DNS do not match, the IP address SHOULD be sent as the `hostname` and the unverified `hostname` SHOULD NOT be sent (see [Security Considerations](#security-considerations)). If the connection fails to the IRC server, the WebIRC gateway MUST return an `ERROR` with an appropriate message and terminate the connection.

Note: Previous implementations referred to the `gateway` field as `user`. This change is for documentation clarity only and maintains compatibility with all existing implementations.

### Options

The `options` parameter is a space-separated set of additional arguments, each argument having the form `<name>[=<value>]`.

The option values are escaped using the same escaping method as [message tags values](../extensions/message-tags.html#escaping-values). It is acknowledged that some of the escaping rules are not strictly required, but the same escaping method is used for consistency.

These options are defined and may be sent by clients while connecting:

- `secure`: This flag indicates that the client has a TLS-secured connection to the gateway. Servers MUST ONLY treat the connection as secure if this flag is sent and the connection from the gateway to the server is also secure with TLS.
- `remote-port=<port>`: This flag indicates the remote port the client has connected to the gateway from.
- `local-port=<port>`: This flag indicates the port the gateway accepted the client connection on (e.g. `6697`, `6667`).
- `certfp-<algo>=<fingerprint>`: This flag indicates the tls client certificate fingerprint supplied to the WebIRC gateway by the user's actual client application.
- `spkifp-<algo>=<fingerprint>`: This flag indicates the public key fingerprint for the tls client certificate supplied to the WebIRC gateway by the user's actual client application.

`<algo>` should be the hash algorithm used to produce the fingerprint supplied such as sha256.

Servers MUST be able to handle options that don't currently have defined values gaining values in the future. For example, they MUST treat the options `secure` and `secure=examplevalue123` in exactly the same way.

### Examples

Generic format.

    WEBIRC password gateway hostname ip [:option1 option2...]

IP address resolves to hostname.

    WEBIRC hunter2 ExampleGateway 3-100-51-198.location.example-isp.com 198.51.100.3

IP address does not resolve to hostname.

    WEBIRC hunter2 ExampleGateway 198.51.100.3 198.51.100.3

Secure connection.

    WEBIRC hunter2 ExampleGateway 198.51.100.3 198.51.100.3 secure

Secure connection with ports passed through.

    WEBIRC hunter2 ExampleGateway 198.51.100.3 198.51.100.3 :secure=examplevalue local-port=6697 remote-port=21726

Error from invalid password.

    ERROR :Invalid WebIRC password

## Use Cases

Webchat applications that proxy the connection to the IRC server are the primary intended use case. WebIRC allows network and channel operators to use available moderation and security tools to their full extent by exposing user IP addresses and hosts.

Other use cases include bouncer services that wish to pass user information to the IRC server.

## Limitations

WebIRC requires that the both WebIRC gateway and IRC server be configured to accept the connection in advance. Specific recommendations are outlined below in the [Security Considerations](#security-considerations) section.

## Security Considerations
WebIRC allows anyone with a valid password to successfully connect to the IRC server and spoof any desired hostname. IRC servers SHOULD use secure passwords and SHOULD whitelist the IP addresses for the allowed WebIRC gateway servers. If the `WEBIRC` command does not originate from a whitelisted IP address or uses an incorrect password, the gateway SHOULD close any connections with an `ERROR` command.

Because the possibility for hostname spoofing exists, IRC servers MAY attempt to further validate or resolve hostnames and match them to an IP address. It is the responsibility of IRC servers to verify the authenticity of connecting users and perform additional security checks as they see fit. To assist network operators and prevent abuse, IRC servers SHOULD show when a WebIRC connection is in use and SHOULD provide the original host when possible. This behavior is non-normative and implementation defined.

## Errata

A previous version of this specification did not include handling the occurrence of an IPv6 address beginning with a colon, which would break the message argument format.
