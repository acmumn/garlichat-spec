GarliChat Protocol
==================

| Common Name    | Cryptographic Scheme | [libsodium](https://libsodium.org) API |
|----------------|----------------------|----------------------------------------|
| encryption key | `xsalsa20poly1305`   | `secretbox`                            |
| ephemeral key  | `x25519blake2b`      | `kx`                                   |
| signing key    | `ed25519ph`          | `sign`                                 |

Wire Protocol
-------------

The GarliChat protocol runs over TCP port 23400.

On connection, the client sends the `ClientHandshake` datagram and the server sends the `ServerHandshake` datagram. The client and server then communicate with a series of `SealedDatagram`s.

### Version Comparisons

Both the client and server handshake encode a two-part version number, major then minor.
If the major versions do not match, the client and server should not be treated as compatible.
If the major versions match but the minor versions do not, a client and server should (if the one with the lower version implements sufficient error handling for unknown messages) be compatible.

### Datagram Descriptions

#### `ClientHandshake`

-	The magic signature `"GARLICKYCLIENT"`
-	The protocol major version number as a `u8`, currently 0
-	The protocol minor version number as a `u8`, currently 1
-	The 32-byte client ephemeral public key
-	The 32-byte client signing public key
-	The 64-byte signature of the ephemeral public key

```
      0x00 0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d 0x0e 0x0f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x00 |'G'  'A'  'R'  'L'  'I'  'C'  'K'  'Y'  'C'  'L'  'I'  'E'  'N'  'T' |0x00|0x01| 0x0f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x10 |                                                                               | 0x1f
     |                             Client Ephemeral Key                              |
0x20 |                                                                               | 0x2f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x30 |                                                                               | 0x3f
     |                               Client Public Key                               |
0x40 |                                                                               | 0x4f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x50 |                                   Signature                                   | 0x5f
     |                                                                               |
0x60 |                                                                               | 0x6f
     |                                                                               |
0x70 |                                                                               | 0x7f
     |                                                                               |
0x80 |                                                                               | 0x8f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
      0x80 0x81 0x82 0x83 0x84 0x85 0x86 0x87 0x88 0x89 0x8a 0x8b 0x8c 0x8d 0x8e 0x8f
```

#### `ServerHandshake`

-	The magic signature `"GARLICKYSERVER"`
-	The protocol major version number as a `u8`, currently 0
-	The protocol minor version number as a `u8`, currently 1
-	The 32-byte server ephemeral public key
-	The 32-byte server signing public key
-	The 64-byte signature of the client ephemeral public key concatenated with the server ephemeral public key

```
      0x00 0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d 0x0e 0x0f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x00 |'G'  'A'  'R'  'L'  'I'  'C'  'K'  'Y'  'S'  'E'  'R'  'V'  'E'  'R' |0x00|0x01| 0x0f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x10 |                                                                               | 0x1f
     |                             Server Ephemeral Key                              |
0x20 |                                                                               | 0x2f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x30 |                                                                               | 0x3f
     |                               Server Public Key                               |
0x40 |                                                                               | 0x4f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x50 |                                   Signature                                   | 0x5f
     |                                                                               |
0x60 |                                                                               | 0x6f
     |                                                                               |
0x70 |                                                                               | 0x7f
     |                                                                               |
0x80 |                                                                               | 0x8f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
      0x80 0x81 0x82 0x83 0x84 0x85 0x86 0x87 0x88 0x89 0x8a 0x8b 0x8c 0x8d 0x8e 0x8f
```

#### `SealedDatagram`

-	A `u64le` length field
-	A 24-byte random nonce
-	An encrypted Update Protocol datagram, whose length is specified by the length field

The Update Protocol datagrams are encrypted with the encryption key derived from the ephemeral keys.

```
      0x00 0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d 0x0e 0x0f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x00 |                Length                 |                 Nonce                 | 0x0f
     +----+----+----+----+----+----+----+----+                                       |
0x10 |                                                                               | 0x1f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x20 |                                                                               | 0x2f
    ...                      Encrypted Update Protocol Datagram                     ...
     |                                                                               |
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
```

Update Protocol
---------------

A client sends `ClientUpdate`s while a server sends `ServerUpdate`s. The `ClientUpdate` and `ServerUpdate` datagrams are used for maintaining a synchronized insert-only set between the client and server.

When a `ServerUpdate` is received, the set of pairs should be unioned into the local set. When an insert is made locally, a `ClientUpdate` should be sent.

### Datagram Descriptions

#### `ClientUpdate`

-	An 8-byte sync token
-	Some number of:
	-	The sender's 32-byte signing public key
	-	A 64-byte signature of the nonce, length, and message
	-	A `u64le` containing the length of the message
	-	A 24-byte random nonce
	-	A Message Protocol datagram whose length is specified by the length field

The sync token must be the most recently received sync token, or 0 if none has been received.

```
      0x00 0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d 0x0e 0x0f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x00 |               Sync Token              |           Sender Public Key           | 0x0f
     +----+----+----+----+----+----+----+----+                                       |
0x10 |                                                                               | 0x1f
     |                                       +----+----+----+----+----+----+----+----+
0x20 |                                       |                                       | 0x2f
     +----+----+----+----+----+----+----+----+                                       |
0x30 |                                                                               | 0x3f
     |                                                                               |
0x40 |                                    Signature                                  | 0x4f
     |                                                                               |
0x50 |                                                                               | 0x5f
     |                                       +----+----+----+----+----+----+----+----+
0x60 |                                       |                Length                 | 0x6f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x70 |                 Nonce                                                         | 0x7f
     |                                       +----+----+----+----+----+----+----+----+
0x80 |                                       |                Message                | 0x8f
     +----+----+----+----+----+----+----+----+                                       |
     |                                                                               |
    ...                                                                             ...
     |                                                                               |
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
     |                           Additional Signed Messages                          |
    ...                                                                             ...
     |                                                                               |
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
```

#### `ServerUpdate`

-	An 8-byte sync token
-	Some number of:
	-	A `i64le` containing a timestamp in milliseconds since the Unix epoch
	-	The sender's 32-byte signing public key
	-	A 64-byte signature of the nonce and message
	-	A `u64le` containing the length of the message
	-	A 24-byte random nonce
	-	An arbitrary message whose length is specified by the length field

The sync token must be treated by the client as opaque.

```
      0x00 0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d 0x0e 0x0f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x00 |               Sync Token              |               Timestamp               | 0x0f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x10 |                               Sender Public Key                               | 0x1f
     |                                                                               |
0x20 |                                                                               | 0x2f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x30 |                                                                               | 0x3f
     |                                                                               |
0x40 |                                    Signature                                  | 0x4f
     |                                                                               |
0x50 |                                                                               | 0x5f
     |                                                                               |
0x60 |                                                                               | 0x6f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
0x70 |                 Length                |                 Nonce                 | 0x7f
     +----+----+----+----+----+----+----+----+                                       |
0x80 |                                                                               | 0x8f
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
     |                                                                               |
    ...                                   Message                                   ...
     |                                                                               |
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
     |                                                                               |
    ...                  Additional Signed and Timestamped Messages                 ...
     |                                                                               |
     +----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+----+
```

Message Protocol
----------------

Every "datagram" in the Message Protocol is a CBOR value.

**Note:** JSON syntax is used here, but the actual values are of course CBOR.

If a CBOR message is received that is not understood by the client, it may be displayed to the user, or ignored.

### Rich Messages

Most messages that include text content represent the text content not as a string, but instead as a `RichMessage`.

A `RichMessage` is an array of `RichMessageComponent`s, where a `RichMessageComponent` is one of:

 - A string, which is interpreted as literal text
 - An object with exactly two keys:
   - `"type"`, whose value is `"user"`
   - `"value"`, whose value is the Base64'd (with the encoding specified in [Section 5 of RFC4648](https://tools.ietf.org/html/rfc4648#section-5)) public key

If an unrecognized or invalid structure is found as a `RichMessage` or `RichMessageComponent`, it should be either ignored or displayed as JSON to the user.

### Datagram Descriptions

#### `Message`

A text message sent by the user.

Encoded as a `RichMessage`.

##### Example

```json
[ "Hello, world!" ]
```

#### `Me`

A text message sent by the `/me` command.

Encoded as an object with exactly two keys:

-	`"type"`, whose value is `"me"`
-	`"value"`, whose value is a `RichMessage`

##### Example

```json
{ "type": "me"
, "value": [ "implemented this spec perfectly" ]
}
```
