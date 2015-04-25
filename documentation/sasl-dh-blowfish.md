---
title: IRCv3 SASL DH-BLOWFISH Authentication Mechanism
layout: spec
---
# SASL DH-BLOWFISH Authentication Mechanism

This specification documents the `DH-BLOWFISH` SASL mechanism currently
implemented by various IRC clients and services.`

The mechanism is non-standard, and specific to IRC software. It uses
Diffie-Hellman key exchange to choose a shared encryption key, which is then
used to encrypt the password sent to the server.

The mechanism does not authenticate the server, and is considered insecure against active MITM attacks.

The mechanism does not allow an authorization identity to be specified.

The mechanism is considered deprecated and should be avoided, as it gives a false sense of security to many users.

## Step 1

The server starts by sending a challenge containing the DH parameters _p_, _g_,
and the server's public key, each serialized as two bytes containing the
parameter's length (network byte order), followed by the parameter itself as an
OpenSSL bignum.

    message = bignum(p) . bignum(g) . bignum(pub_key)

## Step 2

The client computes the shared key from the received DH parameters, and uses it
to encrypt the password using Blowfish in ECB mode. The password must be
encoded as UTF-8 and null-terminated.

The client then sends the response consisting of its own DH public key, the
authentication identity as a null-terminated string, and the encrypted
password. Again, the public key is serialized as two bytes containing the
length, followed by the key itself as a bignum.

    raw_credentials = pad(8, cstring(password))

    enc_credentials = blowfish_ecb[shared_key](raw_credentials)

    message = bignum(pub_key) . cstring(authn_id) . enc_credentials

## Step 3

The server computes the same shared key from the DH parameters and client's
public key, and uses it to decrypt the password.
