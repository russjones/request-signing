# Request Signing Recipe Specification

This document describes the initial version of the yet unnamed request signing
recipe for cryptography.io.

Conceptually, the sender takes an HTTP request, a key, current time, and a
nonce and inputs them into a request signing algorithm that generates a
signature. The signature, along with the nonce and timestamp are added to the
original HTTP request which is then delivered. The recipient recomputes the
signature, checks it matches the signature in the request, checks the timestamp
is within an acceptable range and the nonce has not been seen before, if all
of those conditions are met, the request is declared valid.

This specification describes the format and algorithms for:

* **Key generation**: Recommends the size of the key in bytes, encoding, and
cryptographically secure pseudo-random number generator (CSPRNG) that should be
used to generate the key.
* **Nonce generation**: Specifies the size, encoding, and CSPRNG that should be
used to generate the nonce.
* **Message Authentication Code (MAC) Algorithm**: Specifies the construction
of the MAC algorithm that will be used and the hashing function to be used to
construct the MAC.
* **Message construction**: Specifies how the message under the MAC is to be
constructed and the delimiter that is used to separate fields.
* **Nonce storage and comparison**: Specifies how the recipient should store the
nonce and how many nonce values to store.
* **Timestamp verification**: Specifies the format of the timestamp and an
acceptable range of values for the timestamp.
* **MAC comparison**: Specifies how the recipient should compare the computed and
received values of the signature.

## Specification of Individual Components

### Key generation

The following is recommended for key generation. However, a certain amount of
flexibility in the specification is given for usability purposes. For
example, the user may wish to use an existing application API key as the
signing key or the user may wish to prefix the API key in a certain manner to
match the look and feel of the rest of their application. These use cases need
to be supported.

However, the following is the recommended way to generate a key:

1. Obtain 128-bits of raw random material from a CSPRNG. For Linux based or
OS X systems use `/dev/urandom` for random material, for Windows based systems
use `CryptGenRandom`.
2. Use a base16 encoder to encode the raw bytes of the key into ASCII.

In pseudo-code this would look like:

```
key = base16_encode(csprng(128))
```

### Nonce generation

The following is required for nonce generation:

1. Obtain 128-bits of raw random material from a CSPRNG. For Linux based or
OS X systems use `/dev/urandom` for random material, for Windows based systems
use `CryptGenRandom`.
2. Use a base16 encoder to encode the raw bytes of the key into ASCII.

In pseudo-code this would look like:

```
nonce = base16_encode(csprng(128))
```

### MAC algorithm

A keyed-hash message authentication code (HMAC) is the specific form of a MAC
that is to be used for request signing. The underlying cryptographic hash
function that is to be used is SHA512, this construction will be referred to
as HMAC-SHA512.

The input to HMAC-SHA512 is a key and a message. The key will be the key
specified in the *Key Generation* section. The output of HMAC-SHA512 will
be referred to as the *signature*.

### Message construction

The message is the data that is to be authenticated by the HMAC. The following
three fields are required: *timestamp*, *nonce*, *request body* in that order.

The length of the field (ASCII decimal string) followed by the ASCII pipe
symbol `|` (0x7C) will be used as a delimiter to separate fields.

In pseudo-code this would look like:

```python
message_body = len(timestamp)    + '|' + timestamp    + '|' \
               len(nonce)        + '|' + nonce        + '|' \
               len(request_body) + '|' + request_body
```

The following fields can optionally be included: *HTTP Verb*, *Uniform resource
locator (URL)*, and any number of *headers* in that order.

In pseudo-code this would look like:

```python
message_body = len(timestamp)    + '|' + timestamp    + '|' \
               len(nonce)        + '|' + nonce        + '|' \
               len(request_body) + '|' + request_body + '|' \
               len(http_verb)    + '|' + http_verb    + '|' \
               len(http_uri)     + '|' + http_uri     + '|' \
               len(http_header)  + '|' + http_header
```

### Nonce storage and comparison

A cache to store nonce values that have been seen is required. The size of the
cache is limited to some maximum size to ensure that the nonce cache itself
does not consume all available memory. The maximum size is limited via
supporting a time to live (TTL) mechanism. The size itself is determined by
the acceptable range of timestamp values and the maximum number
of authentication requests accepted.

For example, if the acceptable range of timestamp values is the past five
minutes and maximum acceptance rate is 100/minute, the nonce cache should be
able to hold at least 500 nonce values not accounting for clock skew (see
the *Timestamp verification* section).

### Timestamp verification

Timestamp is the number of seconds since January 1, 1970 UTC from when the
timestamp was created.

For example, `Monday, 14-Sep-15 18:58:10 UTC` would translate to `1442257090`.

Tolerance of clock skew is also required to ensure that computers with clocks
that are slightly out of sync can communicate with each other.

### MAC comparison

The use of a constant time comparison function is required when checking the
value of a computed MAC against the received MAC.

## Implementation

Below we describe how the above components are used by the sender in request
signing and the recipient in request verification.

### Request Signing

1. Read in the key that will be used to sign the request.
1. Read in the parts of the request to be signed.
1. Generate a nonce and save this value.
1. Generate a timestamp and save this value.
1. Generate the message to be signed by joining the parts to be signed with a
delimiter.
1. Generate a signature by inputting the the message and key into HMAC-SHA512.
1. Add the signature, timestamp, and nonce values generated above back into the
HTTP request.
1. Send the HTTP request.

### Request Verification

1. Read in the key that will be used to verify the request. It must be the
same key as used to sign the request.
1. Read in the parts of the request to be verified.
1. Extract the signature, nonce, and timestamp headers.
1. Generate a signature by inputting the message and key into HMAC-SHA512.
1. Verify the signature by using a constant time compare function to compare
the computed signature against the one sent in the HTTP request.
1. Verify the timestamp is within an acceptable range of values.
1. Verify that the nonce has not been seen before.
