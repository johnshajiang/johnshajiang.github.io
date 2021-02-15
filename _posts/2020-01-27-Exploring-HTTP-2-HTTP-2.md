---
layout: post
title: "Exploring HTTP 2: HTTP 2"
---

HTTP/2 protocols contain two RFC: Hypertext Transfer Protocol Version 2 (RFC7540), namely HTTP/2; HPACK: Header Compression for HTTP/2 (RFC7541). RFC7540 states the HTTP/2 semantics, however RFC7541 describes the formats on compressing headers. This article just touches HTTP/2 protocol. The later installment in this series will explore HPACK.

### 1. The problems
HTTP/1.0 allows only one request in a TCP connection. Although HTTP/1.1 introduces streamline to allow multiple requests in a single request, the order of the responses have to be in accordance with the order of the corresponding requests. If the former responses take much time, they will block the later responses. This is so called head-of-line blocking problem. In order to reducing latency, one client has to launch multiple connections for sending requests concurrently. Obviously, this way will bring more load to the server side. And headers generally contain a lot of information, like cookies, which would consume more bandwidth and increase the overhead.

HTTP/2 allows to send and receive multiple requests and responses interleavedly in a single TCP connection. And it introduces a high effective coding format for compressing headers. HTTP/2 can set priorities for the requests that the higher prior requests could be responded quickly. With this newer protocol, server can send responses proactively without client requests. This feature, so-called server push, could reduce unnecessary client requests.

In short, HTTP/2 resolves the performance issues on the prior HTTP versions.

### 2. Initiate HTTP/2
HTTP/2 still uses the same URI schemes, namely http and https, as HTTP/1 does. And HTTP/2 servers don't use different ports to support HTTP/1 and HTTP/2 respectively. This way would upgrade HTTP/1 to HTTP/2 smoothly. It's obvious that the existing web services support HTTP/1 only. If the newer HTTP version uses different scheme or port, the upgrading would take much long time and the cost would be quite high. But HTTP/2 protocol defines two tokens for schemes http and https. One is `h2`, and the other is `h2c` (here "c" stands for cleartext). This article will use `h2c` for the HTTP/2 over TCP directly, however `h2` for the HTTP/2 over TLS.

Then, how does a HTTP/2-compliant client known if the peer supports this same HTTP version? 

For h2c, a HTTP/2-compliant client can use Upgrade header supported by HTTP/1.1 to request the server to upgrade the HTTP version. The request format looks like the below:
```
GET / HTTP/1.1
Host: server.example.com
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c
HTTP2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>
```
`HTTP2-Settings` is a BASE64-encoded string. It's the payload in the client's `SETTINGS` frame.

If the server supports HTTP/2, it should respond "101 Switching Protocols" for accepting the upgrade. The response looks like the followings:
```
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: h2c
```
If the server doesn't support this new protocol, it just ignore the `Upgrade` header and still uses HTTP/1.1.

For `h2`, the two endpoints have to use Transport Layer Security (TLS) Application-Layer Protocol Negotiation Extension (RFC7301), or TLS-ALPN, to negotiate the HTTP version. In theory, TLS-ALPN can be used for negotiating other application protocols.

When both endpoints agree to use HTTP/2, they need to send a connection preface for confirmation.

After the client receives the server's `101 Switching Protocols` response (for `h2c`) or the first application data in TLS connection (for `h2`), it sends the preface immediately. The preface starts with `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` (the hex is `0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a`), and a `SETTINGS` frame (even it's empty) must follow up.

The preface on the server side consists of a `SETTINGS` frame. This frame must be the first frame sent from the server side. This `SETTINGS` frame can be empty, or contain some configuration information that the client should known.

### 3. Frames
HTTP/2 message is in binary format. Compared with text format, it's more effective on processing messages. The smallest unit of HTTP/2 messages is frame. A frame consists of a head and a payload. The length must be multiple 8-bit (octet or byte).
A frame head contains the following fields:
* Length: this field indicates the length of the payload. It is 24-bit unsigned integer.
* Type: this field indicates the type of the frame. It's 8-bit length.
* Flags: this field defines one or more flags. It's 8-bit length. Some frames don't have this field.
* R: this field is a reserved bit. The semantic on this bit isn't defined. The value of this bit must be 0, however when read a frame, this bit just be ignored.
* Stream id: this field declares the identifier of the stream that the frame is in. It's 31-bit length.
* After frame head, it just be payload. The structure and content of a payload is determined by the frame type.

HTTP/2 defines 10 different frames as the followings.
* DATA: one or more `DATA` frames could be used as payload of a request or response.
* HEADERS: it can be used to open a new stream, and carry a header block fragment. A header block consists of a `HEADERS`/`PUSH_PROMISE` frame and arbitrary subsequent `CONTINUATION` frames. These frames can be segmented into arbitrary byte sets. Every such byte set just be called a header block fragment. Except for `HEADERS`/`PUSH_PROMISE` and `CONTINUATION` frames, any other frame cannot be contained by a header block fragment.
* PRIORITY: it is used to setup the priorities of streams.
* RST_STREAM: it's used to terminate streams. If some fatal errors occur or need to cancel a stream, this frame just be sent.
* SETTINGS: this frame carries the configuration parameters used by peer communication. It has a `ACK` flag, which indicates if the parameters are acknowledged by the peer. If an endpoint receives a `SETTINGS` frame with `ACK` flag 0, it should apply the updated parameters as soon as possible.
* PUSH_PROMISE: the server uses this frame to infomate the client that a new stream would be created. This frame is used for implementing one of the well-known features server push.
* PING: this framed is used to test the minimum round-trip time between the peers. It's similar to the ping command, which tests if a connection is available.
* GOAWAY: it's used to initiate shutdown of a connection or to signal serious error conditions. This frame allows an endpoint to gracefully close a connection after all existing streams are processed. There's a competition that if one endpoint creates new stream and simultaneously the peer sends `GOAWAY` frame. In order to dealing with this case, the `GOAWAY` sender has to carry the id of the last stream created by the receiver. After `GOAWAY` is delivered, the sender will ignore all frames from the peer's streams which ids are greater than the aforementioned given id.
* WINDOW_UPDATE: this frame is used in flow control. Its payload consists of a reserved bit and 31-bit unsigned integer. This integer informs the receiver that the sender can transmit in addition to the existing flow-control window.
* CONTINUATION: it is used to continue a sequence of header block fragments. Arbitrary `CONTINUATION` frames can be sent, as long as the preceding frame is on the same stream and is a `HEADERS`, `PUSH_PROMISE`, or `CONTINUATION` frame without the `END_HEADERS` flag set.
The payloads in some frame types, including `DATA`，`HEADERS` and `PUSH_PROMISE`, possibly contain paddings. The padding is useless in business, but it's necessary for security concerns. For example, obscuring the exact size of frame content and mitigating specific attacks within HTTP.

### 4. Stream
A stream is a communication channel between server and client sides. Multiple streams are permitted in a TCP connection, as shown as the below diagram, 
```
+--------+                               +--------+
|        |          Connection           |        |
|        | ============================= |        |
|        |    ----------------------- <-- Stream  |
|        |    +-----+ +---------+ +-+    |        |
|        |    |     | |         | | |    |        |
|        |    +-----+ +---------+ +-+ <-- Frame   |
|        |    -----------------------    |        |
| Client |                               | Server |
|        |    -----------------------    |        |
|        |    +--+ +----+ +---------+    |        |
|        |    |  | |    | |         |    |        |
|        |    +--+ +----+ +---------+    |        |
|        |    -----------------------    |        |
|        | ============================= |        |
|        |                               |        |
+--------+                               +--------+
```
The server and client sides can interleavedly transfer frames over different streams in a single connection. The communication in a single stream still comply with the `sending-request-and-waiting-response` mode. Peers can create new streams, share existing streams, and close the streams created by peers. The sequence of the frames in a stream is meaningful. The receiver must treat the frames in order.

Every stream has an identifier that's 31-bit unsigned integer. In a single connection, the identifiers must be unique. The identifiers created by client must be odd numbers; however, the ones created by server must be even numbers. The 0-value identifier cannot be used by any endpoints. It cannot reuse the identifiers in a same connection, even though the streams have been closed. For a long alive connection, the 31-bit length identifiers may not enough. In this case, a new connection must be created.

For the lifecycle of a stream, HTTP/2 protocol defines 7 states: idle, reserved (local), reserved(remote), open, half closed (local), half closed (remote) and closed. When a peer receives or sends header blocks or the frames (`DATA` and `HEADERS`) with flag `RST_STREAM` will transmit the stream states.

Implementing multiplex based on stream may lead to the race on occupying TCP connection. The flow control solution based on `WINDOW_UPDATE` could ensure to avoid the fatal interruption among streams. The flow control acts on two levels: a single stream; the whole connection. Only `DATA` frames are affected by the flow control. The space consumed by all other frames doesn't occupy the flow-control windows. HTTP/2 only defines the structure and semantics on `WINDOW_UPDATE` frame. The protocol implementations can select different flow control algorithms with their favors.

It can setup priorities for streams. A client could set a priority heavy in `HEADERS` frame when create a new stream. After that the priorities still can be changed. If the performance is limited, the frames in streams that have higher priorities would be delivered priorly. The values of the priorities must be between 1 and 256, and the default value is 16. A stream may depend on another stream. The dependencies could form a relation tree. A stream can set itself as a sub-stream of another stream. The push stream used for server push is the sub-stream of the original stream by default, and the priority is also the default value, exactly 16.

### 5. Message exchange
#### 5.1 Request/Response exchange
As aforementioned, HTTP/2 inherits the semantics from HTTP/1, but the syntaxes delivering the semantics are changed.
A HTTP/2 message consists of the following parts:
* Only response message can contain a header block with 1xx response state. This header block consists of a `HEADERS` frame and its subsequent `CONTINUATION` frames.
* A header block. It consists of a `HEADERS` frame and its subsequent `CONTINUATION` frames.
* Zero or multiple `DATA` frames carrying body messages. The chunked body in HTTP/1 is not applicable to HTTP/2 anymore. With this new protocol, a body can consist of multiple `DATA` frames, so a body in HTTP/2 can be trunked naturally.
* Possibly a header block containing trailer. This header block consists of a `HEADERS` frame and its subsequent `CONTINUATION` frames.

Although HTTP/2 uses the same header fields as those in HTTP/1, the field names must be lower-cased. In addition, the messages in HTTP/1 request line and response state lines are divided into pseudo header fields, and these field names start with colon (:).

HTTP/1 request line format is `method request-target HTTP-version`, then the corresponding HTTP/2 pseudo header fields are `:method=method` and `:path=request-target`. But there's no field for `HTTP-version`, and just uses the default value HTTP/2.

HTTP/1 response state line format is `HTTP-version status-code reason-phrase`. The corresponding HTTP/2 pseudo field is: `:status=status-code`. It has neither field for `HTTP-version`, nor for `reason-phrase`. The `reason-phrase` can be determined by the state codes. HTTP/2 obsoletes as more unnecessary bytes as possible.

However, HTTP/2 protocol defines two more pseudo fields:
* :scheme: the scheme in URI. It is not only http or https, but also other values for non-HTTP services.
* :authority: the authority in URI. Consider `scheme://user:password@host:port/path?query#fragment`, here `user:password@host:port` is the authority.
Section 8.1.3 in RFC7540 provides some simple examples to demonstrate how to convert the HTTP/1 messages to the HTTP/2 counterparts.

#### 5.2 Server push
HTTP/2 server push is a special case for `request-response` mode. After a server receives a client request (original request), in order to pushing more content, it automatically generate new requests (push requests). The responses from a server contains not only the response for original request, but also the ones for push requests.

A client can send a `SETTINGS` frame with parameter `SETTINGS_ENABLE_PUSH` to reject server push, or `RST_STREAM` to cancel a server push stream.

