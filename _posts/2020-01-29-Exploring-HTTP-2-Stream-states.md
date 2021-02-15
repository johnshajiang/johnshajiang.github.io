---
layout: post
title: "Exploring HTTP 2: Stream states"
---

The third installment in this series will describe the states of HTTP/2 streams and the state transition.

### Overview
Every HTTP/2 stream always be in a state in its life cycle. When a peer uses a stream to send or receive some specific frames, the state of the stream would be changed.

```
                         +--------+
                 send PP |        | recv PP
                ,--------|  idle  |--------.
               /         |        |         \
              v          +--------+          v
       +----------+          |           +----------+
       |          |          | send H /  |          |
,------| reserved |          | recv H    | reserved |------.
|      | (local)  |          |           | (remote) |      |
|      +----------+          v           +----------+      |
|          |             +--------+             |          |
|          |     recv ES |        | send ES     |          |
|   send H |     ,-------|  open  |-------.     | recv H   |
|          |    /        |        |        \    |          |
|          v   v         +--------+         v   v          |
|      +----------+          |           +----------+      |
|      |   half   |          |           |   half   |      |
|      |  closed  |          | send R /  |  closed  |      |
|      | (remote) |          | recv R    | (local)  |      |
|      +----------+          |           +----------+      |
|           |                |                 |           |
|           | send ES /      |       recv ES / |           |
|           | send R /       v        send R / |           |
|           | recv R     +--------+   recv R   |           |
| send R /  `----------->|        |<-----------'  send R / |
| recv R                 | closed |               recv R   |
`----------------------->|        |<----------------------'
                         +--------+

   send:   endpoint sends this frame
   recv:   endpoint receives this frame

   H:  HEADERS frame (with implied CONTINUATIONs)
   PP: PUSH_PROMISE frame (with implied CONTINUATIONs)
   ES: END_STREAM flag
   R:  RST_STREAM frame
```
As shown as the above diagram, HTTP/2 defines 7 states for the whole life cycle of a stream. They are `idle`，`reserved (local)`，`reserved (remote)`，`open`，`half closed (local)`，`half closed (remote)` and `closed`. After a peer sends or receives a header block (which consists of a `HEADERS` or `PUSH_PROMISE` frame and its consequent `CONTINUATION` frames) or `RST_STREAM` stream, or a frame (generally `HEADERS` or `DATA`) with `END_STREAM` flag, then the stream state would be changed.

Because of network latency, at one moment, the peers may view a same stream in different states. For example, after a peer uses an `idle` stream to send `HEADERS` frame without flag `END_STREAM`, this peer just thinks this stream is already in state `open`. However, the other peer may not receive this `HEADERS`, so it just thinks the same stream is still in `idle` state.

### idle
Every stream must be in `idle` when it just be initialized. This stream only be allowed to send `HEADERS` frame and receive `HEADERS` and `PRIORITY` frames. After a peer sends or receives `HEADERS` frame, the stream is transmitted to `open` for this peer. But the state won't changed if receiving `PRIORITY` frame.

Please note that an `idle` stream can be reserved by a peer sends or receives `PUSH_PROMISE` frame. This reserved stream would be used for server push, and then the state of this stream goes to `reserved (local)` or `reserved (remote)`.

### open
An `open` stream can send any frame. After it sends or receives a `HEADERS` or `DATA` with `END_STREAM` flag, the state is changed to `half closed (local)` or `half closed (remote)`. If this stream sends or receives `RST_STREAM` frame, it just goes to `closed`.

### half closed (local/remote)
A `half closed (local)` stream can send only `WINDOW_UPDATE`，`PRIORITY` and `RST_STREAM` frames, but it can receive any frame. Accordingly, a `half closed (remote)` frame can receive only `WINDOW_UPDATE`，`PRIORITY` and `RST_STREAM` frames, but it can send any frame.

### reserved (local/remote)
Server sends `PUSH_PROMISE` frame to reserve an `idle` stream for future pushing. And then this stream is in state `reserved (local)` for server; `reserved (remote)` for client. Server sends `HEADERS` to client with this stream. This `HEADERS` is the push response's headers. After sending `HEADERS`, this stream is in `half closed (remote)` from server's view angle. Similarly, after client receives the push response's headers, it just view this stream as `half closed (local)`. Furthermore, only `half closed (local)` or `half closed (remote)` stream can be used to send the push response's body, exactly `DATA` frame.

### closed
After a peer sends or receives `RST_STREAM` frame or a frame with flag `END_STREAM` via `half closed (local)` or `half closed (remote)` stream, the peer will view this stream as `closed`.

`closed` is the final state for a stream. Generally, a `closed` stream can send or receive only `PRIORITY` frame. There's an exception. After a `half closed (local)` or `half closed (remote)` stream sends or receives a `HEADERS` or `DATA` frame with `END_STREAM` flag, although the stream becomes `closed`, it can receives `WINDOW_UPDATE` or `RST_STREAM` in a short time.

