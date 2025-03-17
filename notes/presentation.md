# HTTP/3 QUIC

Prior versions have downsides:
- 1.1 - whitespace, no multiplexing
- 2 - multiplexing capabilities added but can't handle lost or reordered packets -> stalling

The QUIC transport protocol incorporates stream multiplexing and per-stream flow control
and also incorporates TLS 1.3 ([TLS]) at the transport layer.

HTTP/3 relies on QUIC to provide confidentiality and integrity protection of data;
peer authentication; and reliable, in-order, per-stream delivery. While delegating
stream lifetime and flow-control issues to QUIC, a binary framing similar to the
HTTP/2 framing is used on each stream.

Once a client knows that an HTTP/3 server exists at a certain endpoint, it opens a QUIC connection.

Within each stream, you send frames.
Frames can be headers, data, for the HTTP communication, or can also change the connection
through a dedicated control stream.

Each request-response pair consumes a single QUIC stream. So, blocked streams have no effect on others.

Server Push allows servers to initiate a response to something the client hasn't even requested yet.
It is a prediction thing that allows for efficiency

HTTP/3 has its own compression schema since HPACK doesn't work with QUIC.
QPACK uses separate unidirectional streams to modify and track field table
state, while encoded field sections refer to the state of the table without
modifying it.

Servers MAY serve HTTP/3 on any UDP port; an alternative service advertisement always includes an explicit port, and URIs contain either an explicit port or a default port associated with the scheme.

# Connection Establishment
- TLS >v1.3
- if domain, send Server Name Indication (SNI)
- HTTP/3 support is indicated by selecting the ALPN token "h3" in the TLS handshake
- QUIC settings are exchanged in the handshake. HTTP/3 settings are exchanged in SETTINGS frames.

HTTP/3 connections are persistent across multiple requests. For best
performance, it is expected that clients will not close connections until it is
determined that no further communication with a server is necessary (for
example, when a user navigates away from a particular web page) or until the
server closes the connection.

When either endpoint chooses to close the HTTP/3 connection, the terminating
endpoint SHOULD first send a GOAWAY frame (Section 5.2) so that both endpoints
can reliably determine whether previously sent frames have been processed and
gracefully complete or terminate any necessary remaining tasks.

A server that does not wish clients to reuse HTTP/3 connections for a
particular origin can indicate that it is not authoritative for a request by
sending a 421 (Misdirected Request) status code in response to the request

# HTTP Semantics w/ QUIC

A client sends an HTTP request on a request stream, which is a client-initiated bidirectional QUIC stream; see Section 6.1. A client MUST send only a single request on a given stream. A server sends zero or more interim HTTP responses on the same stream as the request, followed by a single final HTTP response

Pushed responses are sent on a server-initiated unidirectional QUIC stream. A server sends zero or more interim HTTP responses, followed by a single final HTTP response

An HTTP message (request or response) consists of:
- the header section, including message control data, sent as a single HEADERS frame,
- optionally, the content, if present, sent as a series of DATA frames,
- and optionally, the trailer section, if present, sent as a single HEADERS frame.

Receipt of an invalid sequence of frames MUST be treated as a connection error of type H3_FRAME_UNEXPECTED.

HTTP messages carry metadata as a series of key-value pairs called "HTTP fields"

HTTP/3 does not use the Connection header field to indicate connection-specific fields; in this protocol, connection-specific metadata is conveyed by other means.

An HTTP/3 implementation MAY impose a limit on the maximum size of the message header it will accept on an individual HTTP message. A server that receives a larger header section than it is willing to handle can send an HTTP 431

Like HTTP/2, HTTP/3 employs a series of pseudo-header fields, where the field name begins with the : character (ASCII 0x3a). These pseudo-header fields convey message control data

Request pseudo-header
:method
:scheme
:authority
:path

Response pseudo-header
:status

The CONNECT method requests that the recipient establish a tunnel to the
destination origin server identified by the request-target

In HTTP/1.x, CONNECT is used to convert an entire HTTP connection into a tunnel
to a remote host. In HTTP/2 and HTTP/3, the CONNECT method is used to establish
a tunnel over a single stream.

All DATA frames on the stream correspond to data sent or received on the TCP
connection. The payload of any DATA frame sent by the client is transmitted by
the proxy to the TCP server; data received from the TCP server is packaged into
DATA frames by the proxy.

Server Push
- interaction mode that permits a server to push a request-response exchange to
  a client in anticipation of the client making the indicated request.
  This trades off network usage against a potential latency gain.

The push ID is used in one or more PUSH_PROMISE frames that carry the control
data and header fields of the request message. These frames are sent on the
request stream that generated the push.

The push ID is then included with the push stream that ultimately fulfills
those promises. The push stream identifies the push ID of the promise that it
fulfills, then contains a response to the promised request as described in
Section 4.1.

the push ID can be used in CANCEL_PUSH frames; see Section 7.2.3. Clients use
this frame to indicate they do not wish to receive a promised resource. Servers
use this frame to indicate they will not be fulfilling a previous promise.

Due to reordering, push stream data can arrive before the corresponding
PUSH_PROMISE frame. When a client receives a new push stream with an
as-yet-unknown push ID, both the associated client request and the pushed
request header fields are unknown. The client can buffer the stream data in
expectation of the matching PUSH_PROMISE. The client can use stream flow
control (Section 4.1 of [QUIC-TRANSPORT]) to limit the amount of data a server
may commit to the pushed stream. Clients SHOULD abort reading and discard data
already read from push streams if no corresponding PUSH_PROMISE frame is
processed in a reasonable amount of time.

# Connection Closure

Idle connections get closed due to idle timeout

Sending a GOAWAY frame allows client and server to agree which requests and pushes were accepted.

Servers send GOAWAYS to tell the client which request number or above will not be handled. Servers can only send this
in decreasing order.

once all requests/pushes are handled, either connection goes idle, or gets shutdown immediately

Fun Fact: 2^62 - 1 and 2^62 - 4
"If a client has consumed all available bidirectional stream IDs, the server
need not send a GOAWAY frame, since the client is unable to make further
requests"

Immediate Closure (CONNECTION_CLOSE frame)

Transport Closure
- an error at the transport level could trigger a closure

# Stream Mapping and Usage

QUIC streams can be either unidirectional, carrying data only from initiator to
receiver, or bidirectional, carrying data in both directions. Streams can be
initiated by either the client or the server.

HTTP does not need to do any separate multiplexing when using QUIC: data sent
over a QUIC stream always maps to a particular HTTP transaction or to the
entire HTTP/3 connection context

All client-initiated bidirectional streams are used for HTTP requests and
responses. A bidirectional stream ensures that the response can be readily
correlated with the request. These streams are referred to as request streams

Unidirectional streams, in either direction, are used for a range of purposes. The purpose is
indicated by a stream type, which is sent as a variable-length integer at the
start of the stream. The format and structure of data that follows this integer
is determined by the stream type.
- Control Streams
- Push Streams
- 2 types from QPACK

Control Streams

stream type of 0x00

Each side MUST initiate a single control stream at the beginning of the
connection and send its SETTINGS frame as the first frame on this stream.

Only one control stream per peer is permitted

Because the contents of the control stream are used to manage the behavior of
other streams, endpoints SHOULD provide enough flow-control credit to keep the
peer's control stream from becoming blocked.

Server push is an optional feature introduced in HTTP/2 that allows a server to
initiate a response before a request has been made. See Section 4.6 for more
details.A push stream is indicated by a stream type of 0x01, followed by the
push ID of the promise that it fulfills, encoded as a variable-length integer.
The remaining data on this stream consists of HTTP/3 frames, as defined in
Section 7.2, and fulfills a promised server push by zero or more interim HTTP
responses followed by a single final HTTP response, as defined in Section 4.1.
Server push and push IDs are described in Section 4.6.

`Only servers can push;

Stream types of the format 0x1f * N + 0x21 for non-negative integer values of N
are reserved to exercise the requirement that unknown types be ignored. These
streams have no semantics, and they can be sent when application-layer padding
is desired. They MAY also be sent on connections where no data is currently
being transferred.

# Frame Layout + Info

When a client sends a CANCEL_PUSH frame, it is indicating that it does not wish to receive the promised resource.

A server sends a CANCEL_PUSH frame to indicate that it will not be fulfilling a promise that was previously sent.

The SETTINGS frame (type=0x04) conveys configuration parameters that affect how
endpoints communicate, such as preferences and constraints on peer behavior.
Individually, a SETTINGS parameter can also be referred to as a "setting"; the
identifier and value of each setting parameter can be referred to as a "setting
identifier" and a "setting value".
- apply to entire connection
- A SETTINGS frame MUST be sent as the first frame of each control stream (see
  Section 6.2.1) by each peer, and it MUST NOT be sent subsequently.


