---
title: "Bi‑directional HTTP over Streams"
abbrev: "bhttp‑Streams"
category: info

docname: draft-vasters-bhttp-streams
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
- bidirectional http
- websocket
- stream framing
- agent communication
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Your Name Here
    organization: Your Organization Here
    email: your.email@example.com

normative:
  RFC2119:
  RFC8174:
  RFC9112:
  RFC6455:
  RFC7838:
  RFC8441:
  RFC9000:
  RFC9220:

informative:
  RFC9113:
  RFC9114:

---

*Bi‑directional HTTP over Streams* (bHTTP‑Streams) defines a minimal framing
mechanism that enables both endpoints of a single, long‑lived, reliable byte
stream to exchange HTTP/1.1 messages in either direction.  The framing layer
does not alter HTTP semantics and introduces no new methods, headers, or
status codes.  Two normative transport bindings are specified: WebSocket and
raw TCP.  A Generic‑Stream profile reuses the TCP binding for any ordered
full‑duplex byte stream.  Optional discovery facilities allow endpoints to
upgrade to bHTTP‑Streams; otherwise the connection remains plain HTTP.

---

# Introduction

HTTP's traditional client‑server model restricts request initiation to one
side of the connection.  Modern agent and server‑to‑server workflows require a
symmetric model where either peer can originate requests.  bHTTP‑Streams
accomplishes this by borrowing WebSocket's frame structure while retaining
unaltered HTTP/1.1 semantics {{RFC9112}}.

The goal mirrors HTTP/2 and HTTP/3: replace only the connection/framing layer
while leaving the semantics untouched.  Compared with HTTP/2 multiplexing,
bHTTP‑Streams remains strictly FIFO per direction and avoids stream IDs,
providing a lowest‑friction upgrade path for simple bidirectional scenarios.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

*Plain HTTP mode* — classic request–response over HTTP/1.1.

*bHTTP mode* — operation after successful upgrade to bHTTP‑Streams.

# Message Model

A complete HTTP message consists of

* a **header block**: request‑line or status‑line plus headers, UTF‑8 encoded,
  terminated by CRLF CRLF;
* a **body** indicated by either `Content‑Length` or
  `Transfer‑Encoding: chunked`.

# Frame Format

bHTTP‑Streams reuses the data frame structure of WebSocket {{RFC6455}} §5 with
these constraints:

* Masking is never applied.
* FIN bit MUST be 1 (no fragmentation).
* Opcodes permitted: `0x1` (text), `0x2` (binary).
* Control opcodes `0x8`–`0xF` MUST NOT appear.
* Payload‑length encoding (7‑bit/16‑bit/64‑bit) is unchanged.

## Fixed‑Length Bodies

If `Content‑Length` is present, the body is conveyed in **one** binary frame of
that length.  `Content‑Length: 0` omits the binary frame.

## Chunked Bodies

For `Transfer‑Encoding: chunked`, the body is a sequence of binary frames
followed by a zero‑length final chunk.  An optional trailer section MAY be
sent in one text frame immediately after the final chunk, formatted per
{{RFC9112}} §6.5.

# Transport Bindings

## WebSocket Binding

Upgrade request MUST include `Sec‑WebSocket‑Protocol: bhttp-streams`; the server
MUST echo the token.  Header block → text frame, body → binary frame(s),
trailers → text frame.

## TCP Binding

The same frame layout runs directly on a raw TCP connection; no WebSocket
handshake, masking, or control frames.

## Generic‑Stream Binding

Any full‑duplex, ordered byte stream (e.g., TLS, QUIC application stream,
UNIX domain socket) can carry the TCP binding unmodified.

# Capability Advertisement and Optional Upgrade

Endpoints MAY advertise bHTTP‑Streams via one or more of:

* WebSocket sub‑protocol `bhttp-streams`;
* TLS ALPN token `bhttp/1`;
* `Upgrade: bhttp-streams` header;
* `Alt‑Svc: bhttp="…"` record {{RFC7838}}.

If the client elects to upgrade, bidirectional request initiation becomes
available; otherwise the connection remains plain HTTP.

# Use with HTTP/2 and HTTP/3

bHTTP‑Streams is byte‑stream oriented and is not carried directly within
HTTP/2 or HTTP/3 request streams.  Integration options:

* **Extended CONNECT WebSocket** — HTTP/2 {{RFC8441}} / HTTP/3 {{RFC9220}}; the
  resulting WebSocket stream then uses the WebSocket binding.
* **QUIC application stream** — if the connection's ALPN is `bhttp/1`, either
  peer may open a non‑HTTP QUIC stream and run the Generic‑Stream binding.

# Error Handling

* Protocol violations (invalid opcode, masking, duplicate `Content‑Length`,
  etc.) → abort connection.
* Application errors are reported with normal HTTP status codes.

# Security Considerations

Confidentiality and integrity are provided by the underlying transport (e.g.,
TLS).  When operating over clear‑text TCP, users SHOULD employ higher‑layer
security mechanisms.  Standard HTTP authentication remains unchanged.

# IANA Considerations

## WebSocket Subprotocol Registration

| Subprotocol   | Reference     |
| ------------- | ------------- |
| bhttp-streams | This document |

## ALPN Registration

| Protocol ID | Reference     |
| ----------- | ------------- |
| bhttp/1     | This document |

---

# Acknowledgments
{:numbered="false"}

The author thanks the HTTPBIS working‑group members for their feedback.