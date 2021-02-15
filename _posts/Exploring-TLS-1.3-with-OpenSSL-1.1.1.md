### TLS 1.3
TLS 1.3, exactly RFC 8446, was published on 10th August 2018. This protocol is a big jump on making TLS faster and stronger.

The well-known features would be:
- 1-RTT full handshake
- Resumption with PSK
- 0-RTT
- Hello retry request
- Post-handshake client authentication
- Key and IV update

And some other important improvements are:
- RSA is obsoleted; Only ECDHE and FFDHE with limited parameters are allowed.
- Only AEAD ciphers, like AES-GCM and ChaCha20_Poly1305, are allowed.
- New cipher suites are introduced, and old ones cannot work.
- PKCS#1v1.5 is removed, instead RSASSA-PSS is preferable.

### OpenSSL 1.1.1
OpenSSL 1.1.1, which was released on 11th September 2018, supports TLS 1.3.

Supported features:
- 1-RTT full handshake
- Resumption with PSK
- 0-RTT
- Hello retry request
- Post-handshake client authentication
- Key and IV update

Supported TLS 1.3 cipher suites:
- TLS\_AES\_256\_GCM\_SHA384
- TLS\_CHACHA20\_POLY1305\_SHA256
- TLS\_AES\_128\_GCM\_SHA256
- TLS\_AES\_128\_CCM\_8\_SHA256
- TLS\_AES\_128\_CCM\_SHA256

Supported named groups:
- secp256r1
- secp384r1
- secp521r1
- x25519
- x448

Obviously, so far only ECDHE, but FFDHE, groups are supported.

In the following sections, this article will explore the most common TLS 1.3 handshaking features with OpenSSL 1.1.1. The OpenSSL build used by the following cases enables SSL trace so that more handshaking details can be unveiled.

### Certificates
Before a client connects to a server via TLS, generally at least a certificate should be prepared. Here designs two RSASSA-PSS certificates: one is self-signed CA, and the other one is server end entity certificate.

```
# Generate self-signed CA
openssl req -x509 -newkey rsa-pss -pkeyopt rsa_keygen_bits:2048 \
  -subj "/CN=ca" -sha256 -nodes -keyout ca.key -out ca.cer

# Generate server end entity certificate
openssl req -newkey rsa-pss -pkeyopt rsa_keygen_bits:2048 \
  -subj "/CN=server" -sha256 -nodes -keyout server.key -out server.csr

openssl x509 -req -CAcreateserial -in server.csr -sha256 -CA ca.cer -CAkey ca.key -out server.cer
```

### 1-RTT Full Handshake
Starting OpenSSL server, namely s_server, is quite simple. Generally, it just needs the certificate, the associated private key and an available port. But this case uses two more options:
- `-groups`, which specifies the supported named groups. Here is `X25519` only.
- `-early_data`, which indicates the server supports 0-RTT and can accept early data from client.

```
openssl s_server -trace -cert server.cer -key server.key -groups X25519 -early_data -accept 9443
```

In order to taking OpenSSL client, namely s_client, connect to a server, the required options include `-CAfile` and `-connect`.
This case just talks TLS 1.3 (`-tls1_3`) and specifies two supported ciphers, namely `TLS_CHACHA20_POLY1305_SHA256` and `TLS_AES_256_GCM_SHA384` (`-ciphersuites`).
In addition, it will receive server session and write it to a local file `sess` (`-sess_out`).

Note that, option `-ciphersuites` is used for TLS 1.3 only, and the cipher suite names, like `TLS_CHACHA20_POLY1305_SHA256`, are standard.
For old TLS versions, option `-ciphers` and short cipher suite names, say `ECDHE-RSA-CHACHA20-POLY1305`, are applied.

```
openssl s_client -trace -CAfile ca.cer -tls1_3 -ciphersuites TLS_CHACHA20_POLY1305_SHA256:TLS_AES_256_GCM_SHA384 \
  -sess_out sess -connect localhost:9443
```

Undoubtedly, this is a full handshake. The first message, exactly `ClientHello`, carries an extension `key_share` that selects the preferable named group and attaches the associated key information.

```
ClientHello, Length=222
  ...
  extensions, length = 143
    ...
    extension_type=key_share(51), length=38
      NamedGroup: ecdh_x25519 (29)
      key_exchange:  (len=32): 88AFF5ADFDC8BA305C358FC92AA7AD7808435D718FB3985D6F4A8FB2CB72F400
```

TLS 1.3 makes a great improvement at this step. Unlike the flows in old protocols, server doesn't wait for client key share in the second round trip any more, so the full handshake could be finished in only one round trip.

The below is a brief summary after handshaking finishes.

```
New, TLSv1.3, Cipher is TLS_CHACHA20_POLY1305_SHA256
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
```

This summary highlights that:
- The session is brand new.
- The negotiated protocol is TLS 1.3.
- The negotiated cipher suite is `TLS_CHACHA20_POLY1305_SHA256`.
- Renegotiation has not been supported.
- Client didn't send early data.

### Resumption with PSK
After handshaking, server could send post-handshake messages, including `NewSessionTicket`.

```
NewSessionTicket, Length=61
  ticket_lifetime_hint=7200
  ticket_age_add=829042100
  ticket_nonce (len=8): 0000000000000000
  ticket (len=32): BA4E09B91059FAF04BB3246ADD91D553AE8274EF3C46ADC53AABABCB2A32D94F
  extensions, length = 8
    extension_type=early_data(42), length=4
      max_early_data=16384
```

The above `NewSessionTicket` is sent by server in the connection in section "Full Handshake". Client saved it to a local file, exactly `sess`. This file will be used for resuming the previous session in a subsequent connection. Option `-sess_in` specifies this local session file.

```
openssl s_client -trace -CAfile ca.cer -tls1_3 -ciphersuites TLS_CHACHA20_POLY1305_SHA256:TLS_AES_256_GCM_SHA384 \
  -sess_in sess -sess_out sess -connect localhost:9443
```

In this connection, `ClientHello` message contains extension `pre_shared_key` (here is `psk`).

```
ClientHello, Length=301
  ...
  extensions, length = 222
    ...
    extension_type=psk(41), length=75
      0000 - 00 26 00 20 94 a2 ba af-ca bd d1 2f 21 77 a2   .&. ......./!w.
      000f - de 42 24 ca 9a e9 10 86-80 dd 17 6b 4b f5 01   .B$........kK..
      001e - a2 d0 3c e9 e4 70 1e 1f-87 57 00 21 20 3c d1   ..<..p...W.! <.
      002d - 16 26 b9 02 86 f3 dc 30-59 eb 05 75 42 ae 48   .&.....0Y..uB.H
      003c - 4e fa 5a 21 99 29 69 2c-af be d6 37 0d 91 a9   N.Z!.)i,...7...
```

Because the server had been authenticated by PSK, so no certificate was sent from server to client. After this handshaking, the session summary looks like,

```
Reused, TLSv1.3, Cipher is TLS_CHACHA20_POLY1305_SHA256
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
```

Only one item, namely `Reused`, is different from the counterpart (`New`) in the connection in section "Full Handshake". That means the session is resumed.

Note that s_server sends two pieces of `NewSessionTicket`. This is allowed by the protocol. But the local session file only saves the last ticket.

### 0-RTT
0-RTT looks like an variant of resumption. Client also requests to resume a previous session, but furthermore it can immediately send application data (early data) before server response.

OpenSSL client can send early data via option `-early_data`. Actually, this option specifies a local file and sends the content as early data.

```
# Generate early data
echo "EARLY_DATA" > data

# Reconnect the server and send early data
openssl s_client -trace -CAfile ca.cer -tls1_3 -ciphersuites TLS_CHACHA20_POLY1305_SHA256:TLS_AES_256_GCM_SHA384 \
  -sess_in sess -early_data data -connect localhost:9443
```

This `ClientHello` contains not only `pre_shared_key`, but also `early_data` extension. After sending `ClientHello`, client immediately sends application data, which is encrypted by the key derived from PSK.

```
Sent Record
Header:
  Version = TLS 1.0 (0x301)
  Content Type = Handshake (22)
  Length = 309
    ClientHello, Length=305
      ...

Sent Record
Header:
  Version = TLS 1.2 (0x303)
  Content Type = ChangeCipherSpec (20)
  Length = 1
    change_cipher_spec (1)

Sent Record
Header:
  Version = TLS 1.2 (0x303)
  Content Type = ApplicationData (23)
  Length = 28
  Inner Content Type = ApplicationData (23)
```

This time session summary indicates not only the session is resumed, but early data was received by server as well.

```
Reused, TLSv1.3, Cipher is TLS_CHACHA20_POLY1305_SHA256
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was accepted
Verify return code: 0 (ok)
```

## Hello Retry Request
In theory, client can send an empty `key_share`. If that, server will select the named group by itself. But server doesn't get client key share yet, then it has to ask client to send key share again via the responded `HelloRetryRequest` message (in fact, it's also `ServerHello` message). Obviously one more round trip has to be costed.

Furthermore, if the named group selected by client is not supported by server, message `HelloRetryRequest` also has to be sent from server, and then ask for additional round trip. This case is demonstrated by the below command,

```
openssl s_client -trace -CAfile ca.cer -tls1_3 -ciphersuites TLS_CHACHA20_POLY1305_SHA256:TLS_AES_256_GCM_SHA384 \
  -groups P-256:X25519 -connect localhost:9443
```

The client limits the supported groups to `P-256` and `X25519`, but `P-256` is the first favourite. The `key_share` extension in message `ClientHello` must select `P-256`.

```
ClientHello, Length=249
  ...
  extensions, length = 170
    ...
    extension_type=key_share(51), length=71
      NamedGroup: secp256r1 (P-256) (23)
      key_exchange:  (len=65): 047D66A4B624DE285B02D7736093824CBBABFCF65444951165A0E909B4C0F33DF85C0A2253ACC37DDB3220F80878657FFA5A1EFDF0AA8C758DEFA6222FAD21492
```

Because server doesn't support this group, but support `X25519` only, so the `key_share` in `ServerHello` message (aka `HelloRetryRequest` here) just selects `X25519`, but without `key_exchange` value.

```
ServerHello, Length=84
  ...
  extensions, length = 12
    ...
    extension_type=key_share(51), length=2
      NamedGroup: ecdh_x25519 (29)
```

Then the client sends `key_share` in another `ClientHello` message and selects `X25519` with `key_exchange` value.

```
ClientHello, Length=216
  extensions, length = 137
    ...
    extension_type=key_share(51), length=38
      NamedGroup: ecdh_x25519 (29)
      key_exchange:  (len=32): 4FD32C9075686176CA099F99BB2CE928666ACA20E4AEEE550D8A4773FDEA8528
```

Don't worry that multiple-round-trips handshaking would be rare case. Generally, client doesn't send an empty `key_share` and server should support the same even more named groups as those client supports.

Of course, the existing (server side) implementations may not address all of defined groups. For example, as aforementioned that OpenSSL doesn't support FFDHE groups yet. TLS 1.3 just was published, and this issue will be resolved as time goes on.

