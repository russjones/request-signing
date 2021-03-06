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
* **Timestamp Format**: Specifies for the format for the timestamp.
* **Message Authentication Code (MAC) Algorithm**: Specifies the construction
of the MAC algorithm that will be used and the hashing function to be used to
construct the MAC.
* **Message construction**: Specifies how the message under the MAC is to be
constructed and the delimiter that is used to separate fields.
* **Nonces and Timestamps**: Specficies how nonces are to be stored and valid
ranges for timestamps.
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

### Timestamp Format

Timestamp is the number of seconds since January 1, 1970 UTC from when the
timestamp was created.

For example, `Monday, 14-Sep-15 18:58:10 UTC` would translate to `1442257090`.

### MAC algorithm

A keyed-hash message authentication code (HMAC) is the specific form of a MAC
that is to be used for request signing. The underlying cryptographic hash
function that is to be used is SHA512, this construction will be referred to
as HMAC-SHA512.

The input to HMAC-SHA512 is a key and a message. The format for the key is
specified in the *Key Generation* section. The format for the message is
specified in *Message construction* section. The output of HMAC-SHA512 will
be referred to as the *signature*.

In pseudo-code this would look like:

```python
signature = hmac.new(key=shared_secret,
                     msg=message,
                     digestmod=hashlib.sha512).hexdigest())
```

### Message construction

The message is the data that is to be authenticated by the HMAC. The following
fields are required: *request verb*, *request URL*, *timestamp*, *nonce*,
*request body* in that order.

The length of the field (ASCII decimal string) followed by the ASCII pipe
symbol `|` (0x7C) will be used as a delimiter to separate fields.

In pseudo-code this would look like:

```python
message_body = len(http_verb)    + '|' + http_verb    + '|' \
               len(http_url)     + '|' + http_url     + '|' \
               len(timestamp)    + '|' + timestamp    + '|' \
               len(nonce)        + '|' + nonce        + '|' \
               len(request_body) + '|' + request_body
```

Optionally, any number of HTTP headers can be included:

In pseudo-code this would look like:

```python
message_body = len(http_verb)    + '|' + http_verb    + '|' \
               len(http_url)     + '|' + http_url     + '|' \
               len(timestamp)    + '|' + timestamp    + '|' \
               len(nonce)        + '|' + nonce        + '|' \
               len(request_body) + '|' + request_body + '|' \
               len(http_header)  + '|' + http_header
```

### Nonces and Timestamps

Nonce values are needed to prevent replay attacks. In a naive implementation,
nonce values would be stored forever to effectively prevent replay attacks.
However this leads to excessive memory consumption to store a near infinite
number of nonce values. To limit the amount of memory needed to store nonce
values, a timestamp is used to restrict the range of acceptable timestamps
(and in turn nonce values) this specification accepts and is paired with a
Time-to-Live (TTL) based cache to automatically expire nonce values older
than said time range.

The default timestamp and TTL value is 100 seconds. However, this parameter is
configurable to allow the user to adjust for network conditions such as slow 
links, large requests, or other problems that are outside of the scope of this
specification.

#### Timestamps

##### Skew and Verification

Tolerance of clock skew is also required to ensure that computers with clocks that
are slightly out of sync can communicate with each other. A clock skew of 5
seconds is the recommended value that implementations should default to, while
still allowing configuration of this parameter for non-standard environments.

When verifying a timestamp, the range of acceptable values must be no greater
than the current time plus the allowable clock skew and no less than the current
time minus the maximum time in the nonce cache plus the allowable clock skew. 

On a time line, this would look like below. The shaded area representing the
acceptable timestamp values.

```
 now - (max_time_in_cache - clock_skew)                       now + clock_skew
             V                                                      V
       |     |▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒|▒▒▒▒▒|
-------|-----|▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒|▒▒▒▒▒|----------
       |     |▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒|▒▒▒▒▒|
       ^                                                      ^  
 (now - max_time_in_cache)                                   now      
```

In pseudo-code this would look like:

```python
if timestamp >= now + clock_skew:
        raise TimestampFromFutureException()

if timestamp <= now - (max_time_in_cache - clock_skew):
        raise TimestampFromPastException()
```

#### Nonce Storage

To implement storage for nonce values, a cache with the following properties
is required:

1. A function that can be used used to check if the value exists in the cache,
and if not, place it in the cache.
1. Automatically age off values older than some TTL.

In pseudo-code this would look like:

```python
class Cache(object):
    def __init__(self, ttl)
        self.cache = {}
        self.ttl = ttl

    def in_cache(nonce):
        if nonce in self.cache:
            return True
    
        self.cache[nonce] = (True, time.time())
        return False

    def get(self, key):
        value, insert_time = self.cache[key]
        object_age = time.time() - insert_time
        if object_age > self.ttl:
            raise KeyError(key)
                    
        delete self.cache[key]
        return value

    def set(self, key, value):
        self.cache[key] = (value, time.time())
```

### MAC comparison

The use of a constant time comparison function is required when checking the
value of a computed MAC against the received MAC.

In pseudo-code this would look like:

```python
if not constant_time.bytes_eq(computed_hmac, received_hmac):
    raise AuthenticationException()
```

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

### Test Vectors

The **first test** is an HTTP request with no extra header included. The following
values are used for the key, nonce, and timestamp:

* Key: `042DAD12E0BE4625AC0B2C3F7172DBA8`
* Nonce: `000102030405060708090a0b0c0d0e0f`
* Timestamp: `1330837567`

```
POST /path?key=value&key=value#fragment HTTP/1.1
Content-Type: application/json

{"hello": "world"}
```

The value of the signature is:
`2d5abe19f48f8bbd5673f3dfefa3c85d297e75c91c15eec2f4cbeafd6c1248e6b2f845dfb10f7f46e79d78309f0c3c2859a83f9c2a06ef4f6acb8cc9a942b33a`

The **second test** is an HTTP request with a extra header also signed. The
following values were used for the key, nonce, and timestamp.

* Key: `6E8D307918B1551C181C3CEAFC40DAAA`
* Nonce: `f5b5d399412bed62bcca4230b4812355`
* Timestamp: `1442257090`

```
POST /foobar HTTP/1.1
Content-Type: application/json
X-Sample-Header: foobarbaz

{"hello": "world"}
```

The value for the signature is:
`2da135db5b8e8c843587fab1a4ddf20f0bc18bb9b379add64cb117a3eba516a1296be09683a0d69d9ec42b1be74f0dc9166550e082e875192b75faff78b2d16a`

### Security Considerations

1. Transport Layer Security (TLS) should be used whenever possible with
this request signing specification. TLS provides confidentiality guarantees
that are otherwise out of scope for this specification, but are necessary to
to prevent an eavesdropper from viewing traffic.

1. A known weakness of this specification occurs when you do not use TLS and
you have multiple hosts validating signed requests. For example, if you have
multiple servers that process HTTP webhooks behind a load balancer. Because
the nonce cache is local to a server, an attacker can potentially capture a
signed request and replay it to the other servers that are yet to see it. To
mitigate this weakness, either TLS should be used or the nonce should be
stored in a shared cache.
