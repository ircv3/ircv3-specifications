---
title: IRCv3 SASL DH-AES Authentication Mechanism
layout: spec
---
# SASL DH-AES Authentication Mechanism

This specification documents the `DH-AES` SASL mechanism currently implemented
by some IRC clients and services.`

The mechanism is non-standard, and specific to IRC software. It uses
Diffie-Hellman key exchange to choose a shared encryption key, which is then
used to encrypt the password sent to the server.

The mechanism does not authenticate the server, and is considered insecure against active MITM attacks. Therefore it may give a false sense of security to users. It does however have improvements over DH-BLOWFISH.

The mechanism does not allow an authorization identity to be specified.

## Step 1

The server starts by sending a challenge containing the DH parameters _p_, _g_,
and the server's public key, each serialized as two bytes containing the
parameter's length (network byte order), followed by the parameter itself as an
OpenSSL bignum.

    message = bignum(p) . bignum(g) . bignum(pub_key)

## Step 2

The client computes the shared key from the received DH parameters, and uses it
to encrypt the credentials using AES-128 in CBC mode with a random IV.

The credentials consist of username (authentication identity) and password
concatenated together; each item must be encoded as UTF-8 and null-terminated.
The result is then padded to 16 bytes.

The client then sends the response consisting of its own DH public key, the IV,
and the encrypted credentials. Again, the public key is serialized as two bytes
containing the length, followed by the key itself as a bignum.

    raw_credentials = pad(16, cstring(authn_id) . cstring(password))

    enc_credentials = aes128_cbc[shared_key, iv](raw_credentials)

    message = bignum(pub_key) . iv . enc_credentials

## Step 3

The server computes the same shared key from the DH parameters and client's
public key, and uses it to decrypt the username and password.
