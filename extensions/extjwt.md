---
title: IRCv3 `extjwt` extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: Darren Whitlen
    period: 2018
    email: darren@kiwiirc.com
---
## Introduction

IRC networks and clients often provide external web hosted services for their user base such as forums, wikis and pastebins. However, without extra development work on the IRC network and the external service they usually require separate user authentication.

This feature provides a way for the external service to verify that a user is connected with specific permissions and / or is joined to any channels, removing the need for any development ties between the IRC network and the external service such as using XMLRPC or custom bots or shared database access.

Once in use, this:
1. Makes it easier to deploy an external service as it does not need access to your IRC server.
2. Improves security in that a misconfigured or vulnerable external service cannot impact the IRC network or user accounts.
3. Provides a common method to verify user status on a channel (joined, channel modes) while using well known libraries and methods that are available for many languages and frameworks (JWT).

### How the verification works

The IRC network generates signed JWT tokens and sends them to the client. The client may use this token to open an external service with the token in its URL.

The JWT token (https://jwt.io/) is a base64 encoded JSON payload that is signed with a secret string. The JSON payload consists of known claims that include:
* `exp` `1529917513` Expiry time for this token.
* `iss` `"irc.example.org"` The server name that generated this token.
* `sub` `"somenick"` The nick of the user that generated this token.
* `vfy` `"https://irc.example.org/extjwtverify/?t=%s"` The URL that can verify this token.
* `account` `"somenick"` The account name of the user that generated this token. Empty if not logged in.
* `umodes` `["o"]` User modes the IRCd wishes to disclose. Eg, if the user is an operator.

When an external service is opened with this token in its URL, the external service verifies that the token has not been tampered with by one of two methods:
1. If the external service has a shared secret for this IRC network, it can now verify the token directly.
2. The external service can make a HTTP GET request to the URL given in the `vfy` claim, replacing `%s` with the token string. The request MUST be responded to with a HTTP 200 status if the token has been verified, or a HTTP 401 status if the token is invalid.

Once successfully verified, the external service can then use the available claims in the token to create any required user accounts and log the user in automatically.

## Usage

If the feature is available on the IRC server, the `EXTJWT=1` token pair is added to its ISUPPORT list where `1` denotes the version of this feature. Currently this is the only version of EXTJWT, version `1`.

Only one new command is introduced in this extension, `EXTJWT`.

### The EXTJWT Command

Syntax: `EXTJWT ( <channel> | * ) [service_name]`

Response syntax: `EXTJWT <requested_target> <service_name> [*] <jwt_token>`

The `service_name` is an optional name for what the token is being requested for. Eg, `EXTJWT * forum.example.org`. The server MAY take this service name to sign its token using different secrets. It defaults to `*` if not provided.

The client will send at minimum `EXTJWT *` to the server to request a new JWT token. The server MUST then reply with the above response syntax with the `requested_target` and `service_name` defaulting to `*` if they were not provided in the request. The JWT token MUST contain the following claims that are relevant to the client at that time:

* `exp` Number; Unix timestamp for when this token expires. See below for notes on the expiry claim.
* `iss` String; The server name that generated this token.
* `sub` String; The nick of the client that requested this token.
* `account` String; The account name of the user that requested this token. Empty if not available.
* `umodes` []String; An array of user modes the IRCd may want to disclose. Eg, if the user is an operator.

Optionaly, the JWT token MAY contain the following claim to provide the external service a way to verify the token:
* `vfy` String; A HTTP URL to verify the token.

If the command sent by the client includes a channel name, Eg. `EXTJWT #channel`, the server MUST then reply with the channel name as its `requested_target` parameter, along with the JWT token containing the above claims and also the following claims relevant to the channel at that time:

* `channel` String; The channel name this token is related to.
* `joined` Number; Unix timestamp of the time at which the client joined the channel. 0 if not joined. 1 if no timestamp is available.
* `cmodes` []String; An array of the channel modes the client has in this channel.

The IRC server MUST include the above claims but MAY include any extra claims.


#### Handling long responses

In some cases the encoded token may be longer than the maximum line length allowed between the client and server. In this case, the third parameter of the response MUST be `*` to indicate that further data will follow. The final chunk of the response sent to the client MUST NOT include this extra `*` parameter.

Eg:
~~~
[C -> S] EXTJWT #channel
[S -> C] EXTJWT #channel * * eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1Mjk5MTc1MTMsImlzcyI6ImlyYy5leGFtcGxlLm9yZyIsIm5pY2siOiJ0ZXN0bmljayIsImFjY291bnQiOiJ0ZXN0bmljayIsInVtb2RlcyI6W10sImNoYW5uZWwiOiIjY2hhbm5lbCIsImpvaW5lZCI6MTUyOTkxNzUwMSwiY21v
[S -> C] EXTJWT #channel * * ZGVzIjpbIm8iXSwiY2xhaW0xIjoic29tZSBsb25nIHZhbHVlIiwiY2xhaW0yIjoic29tZSBsb25nIHZhbHVlIiwiY2xhaW0zIjoic29tZSBsb25nIHZhbHVlIiwiY2xhaW00Ijoic29tZSBsb25nIHZhbHVlIiwiY2xhaW01Ijoic29tZSBsb25nIHZhbHVlIiwiY2xhaW02Ijoic29tZ
[S -> C] EXTJWT #channel * SBsb25nIHZhbHVlIiwiY2xhaW03Ijoic29tZSBsb25nIHZhbHVlIiwiY2xhaW04Ijoic29tZSBsb25nZXIgdmFsdWUgdG8gbWFrZSBzdXJlIHRoaXMgdG9rZW4gaXMgdG9vIGxvbmcgdG8gc2VuZCBvbiBvbmUgSVJDIDUxMiBjaGFyYWN0ZXIgbGluZSJ9.wxRb7lH9OjENg_dTmPrDglBsN3Z17g1eEGJdp9Jsbqg
~~~

#### Notes on the token expiry claim

When generating a token, the expiry (exp) claim must be configured with enough length of time for the token to be used but also be short enough that the token does not last indefinately, leaving the user with a valid token after the user has left a channel or changed its network modes.

30 seconds is usually enough time for the client to receive the token from the `EXTJWT` command and then open the external service webpage, however, non-webpage based services may require a longer expiration depending on its implementation.

## Examples

All examples may be verified using the secret of "your-256-bit-secret".

#### A client logged into the IRC server with operator privileges
~~~
[C -> S] EXTJWT *
[S -> C] EXTJWT * * eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1Mjk5MTc1MTMsImlzcyI6ImlyYy5leGFtcGxlLm9yZyIsInN1YiI6InNvbWVuaWNrIiwiYWNjb3VudCI6InNvbWVuaWNrIiwidW1vZGVzIjpbIm8iXX0.wZbYLX4rgDDB4svLQIsx5jq5_Dc0csdqgamVsgocOas
~~~

Where the replied token is decoded into:
~~~json
{"exp":1529917513,"iss":"irc.example.org","sub":"somenick","account":"somenick","umodes":["o"]}
~~~

#### A client connected to the IRC server without a registered account or operator privileges
~~~
[C -> S] EXTJWT *
[S -> C] EXTJWT * * eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1Mjk5MTc1MTMsImlzcyI6ImlyYy5leGFtcGxlLm9yZyIsInN1YiI6InNvbWVuaWNrIiwiYWNjb3VudCI6IiIsInVtb2RlcyI6W119.i_ak4qvb1BPdH0a0HyRNTz036rHE2lGrZ17SQV3LAFE
~~~

Where the replied token is decoded into:
~~~json
{"exp":1529917513,"iss":"irc.example.org","sub":"somenick","account":"","umodes":[]}
~~~

#### A client logged into the IRC server and has channel operator privileges on a channel
~~~
[C -> S] EXTJWT #channel
[S -> C] EXTJWT #channel * eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1Mjk5MTc1MTMsImlzcyI6ImlyYy5leGFtcGxlLm9yZyIsInN1YiI6InRlc3RuaWNrIiwiYWNjb3VudCI6InRlc3RuaWNrIiwidW1vZGVzIjpbXSwiY2hhbm5lbCI6IiNjaGFubmVsIiwiam9pbmVkIjoxNTI5OTE3NTAxLCJjbW9kZXMiOlsibyJdfQ.A6tYn5w2-yNQ3Ni-W8EMaCmtssc4EG-M_OTI4sf-yUA
~~~

Where the replied token is decoded into:
~~~json
{"exp":1529917513,"iss":"irc.example.org","sub":"testnick","account":"testnick","umodes":[],"channel":"#channel","joined":1529917501,"cmodes":["o"]}
~~~

#### A server responding with a verification (`vfy`) claim
~~~
[C -> S] EXTJWT #channel forum.example.org
[S -> C] EXTJWT #channel forum.example.org eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1Mjk5MTc1MTMsImlzcyI6ImlyYy5leGFtcGxlLm9yZyIsInZmeSI6Imh0dHBzOi8vaXJjLmV4YW1wbGUub3JnL2V4dGp3dD90PSVzIiwic3ViIjoidGVzdG5pY2siLCJhY2NvdW50IjoidGVzdG5pY2siLCJ1bW9kZXMiOltdLCJjaGFubmVsIjoiI2NoYW5uZWwiLCJqb2luZWQiOjE1Mjk5MTc1MDEsImNtb2RlcyI6WyJvIl19.WhZpr-m2v2T6y1tlLFukX3pvk4k-tiQLzlW-poR74fk
~~~

Where the replied token is decoded into:
~~~json
{"exp":1529917513,"iss":"irc.example.org",vfy":"https://irc.example.org/extjwt?t=%s","sub":"testnick","account":"testnick","umodes":[],"channel":"#channel","joined":1529917501,"cmodes":["o"]}
~~~
