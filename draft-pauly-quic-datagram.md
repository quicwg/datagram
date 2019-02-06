---
title: An Unreliable Datagram Extension to QUIC
abbrev: QUIC Datagrams
docname: draft-pauly-quic-datagram-latest
date:
category: std

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: T. Pauly
    name: Tommy Pauly
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: tpauly@apple.com
  -
    ins: E. Kinnear
    name: Eric Kinnear
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: ekinnear@apple.com
  -
    ins: D. Schinazi
    name: David Schinazi
    org: Google LLC
    street: 1600 Amphitheatre Parkway
    city: Mountain View, California 94043
    country: United States of America
    email: dschinazi.ietf@gmail.com

informative:
  I-D.ietf-quic-transport:
  RFC0768:
  RFC2119:
  RFC6347:
  RFC8174:

--- abstract

This document defines an extension to the QUIC transport protocol to
add support for sending and receiving unreliable datagrams over
a QUIC connection.

--- middle

# Introduction

The QUIC Transport Protocol {{I-D.ietf-quic-transport}} provides a secure,
multiplexed connection for transmitting reliable streams of application data.
Reliability within QUIC is performed on a per-stream basis, so some frame
types are not eligible for retransmission.

Some applications, particularly those that need to transmit real-time data,
prefer to transmit data unreliably. These applications can build directly upon
UDP {{RFC0768}} as a transport, and can add security with DTLS {{RFC6347}}.
Extending QUIC to support transmitting unreliable application data would
provide another option for secure datagrams, with the added benefit of sharing
a cryptographic and authentication context used for reliable streams.

This document defines four new DATAGRAM QUIC frame types, which
carry application data without requiring retransmissions.

## Specification of Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{RFC2119}} {{RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Motivation

Transmitting unreliable data over QUIC provides benefits over existing
solutions:

- Applications that open both a reliable TLS stream and an unreliable
DTLS flow to the same peer can benefit by sharing a single handshake
and authentication context between a reliable QUIC stream and flow
of unreliable QUIC datagrams. This can reduce the latency required
for handshakes.

- QUIC uses a more nuanced loss recovery mechanism than the DTLS
handshake, which has a basic packet loss retransmission timer. This
may allow loss recovery to occur more quickly for QUIC data.

- QUIC datagrams, while unreliable, can support acknowledgements,
allowing applications to be aware of whether a datagram was successfully
received.

These reductions in connection latency, and application insight into
the delivery of datagrams, can be useful for optimizing audio/video
streaming applications, gaming applications, and other real-time
network applications.

Unreliable QUIC datagrams can also be used to implement an IP
packet tunnel over QUIC, such as for a Virtual Private Network (VPN).
Internet-layer tunneling protocols generally require a reliable and 
authenticated handshake, followed by unreliable secure transmission
of IP packets. This can, for example, require a TLS connection for
the control data, and DTLS for tunneling IP packets. A single
QUIC connection could support both parts with the use of unreliable
datagrams.

# Transport Parameter

Support for receiving the DATAGRAM frame types is advertised by means
of a QUIC Transport Parameter (name=max_datagram_frame_size, value=0x0020).
The max_datagram_frame_size transport parameter is an integer value
(represented as a variable-length integer) that represents the maximum
size of a DATAGRAM frame (including the frame type, datagram ID, length and
payload) the endpoint is willing to receive, in bytes. An endpoint that
includes this parameter supports the DATAGRAM frame types and is willing to
receive such frames on this connection. Endpoints MUST NOT send DATAGRAM
frames until they have sent and received the max_datagram_frame_size transport
parameter. Endpoints MUST NOT send DATAGRAM frames of size strictly larger
than the value of max_datagram_frame_size the endpoint has received from its
peer. An endpoint that receives a DATAGRAM frame when it has not sent the
max_datagram_frame_size transport parameter MUST terminate the connection with
error PROTOCOL_VIOLATION. An endpoint that receives a DATAGRAM frame that is
strictly larger than the value it sent in its max_datagram_frame_size
transport parameter MUST terminate the connection with error
PROTOCOL_VIOLATION.


# Datagram Frame Type

DATAGRAM frames are used to transmit application data in an unreliable manner.
The DATAGRAM frame type takes the form 0b001100XX (or the set of values from
0x30 to 0x33). The least significant bit of the DATAGRAM frame type is the
LEN bit (0x01). It indicates that there is a Length field present. If this
bit is set to 0, the Length field is absent and the Datagram Data field extends
to the end of the packet. If this bit is set to 1, the Length field is present.
The second least significant bit of the DATAGRAM frame type is the
DATAGRAM_ID bit (0x02). It indicates that there is a Datagram ID field present.
If this bit is set to 0, the Datagram ID field is absent and the Datagram ID is
assumed to be zero. If this bit is set to 1, the Datagram ID field is present.

A DATAGRAM frame is shown below.

~~~
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      [Datagram ID (i)]                      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         [Length (i)]                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Datagram Data (*)                     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #datagram-format title="DATAGRAM Frame Format"}

The fields of a DATAGRAM frame are as follows:

Datagram ID:

: A variable-length integer indicating the datagram ID of the datagram (see
{{datagram-id}}).

Length:

: A variable-length integer specifying the length of the datagram in bytes. If the length
is zero, the data extends to the end of the QUIC packet.

Datagram Data:

: The bytes of the datagram to be delivered.

# Behavior and Usage

When an application sends an unreliable datagram over a QUIC connection,
QUIC will generate a new DATAGRAM frame and send it in the first available
packet. This frame SHOULD NOT be delayed, but MAY be coalesced
with other STREAM or DATAGRAM frames.

When a QUIC endpoint receives a valid DATAGRAM frame, it SHOULD deliver the
data to the application immediately.

DATAGRAM frames MUST be protected with either 0-RTT or 1-RTT keys.

## Datagram Identifiers {#datagram-id}

Since several applications relying on datagrams have the need to identify which
application-level flow a given datagram is a part of, DATAGRAM frames carry a
datagram identifier. Applications that do not have a need for the identifier
can use the value zero on their DATAGRAM frames and use the DATAGRAM_ID bit
to omit sending the identifier over the wire. If an application uses a mixture
of DATAGRAM frames with and without the DATAGRAM_ID bit set, the frames without
it are assumed to be part of the application-level flow with Datagram ID zero.

## Flow Control and Acknowledgements

Although the DATAGRAM frame is not retransmitted upon loss detection,
it does contribute to the maximum data for the overall connection. Packets that
contain only DATAGRAM frames do need to be acknowledged, but
implementations SHOULD defer and batch acknowledgements since
the timing of these acknowledgements is not used for loss recovery.

The DATAGRAM frame does not provide any explicit flow control signaling
apart from the connection-level flow control. DATAGRAM frames are flow
controlled only when the maximum data for the connection is hit, at which point
the BLOCKED frame is sent.

In cases in which a DATAGRAM frame is blocked due to connection-level
flow control or congestion control, an implementation MAY drop the frame
without sending it.

# Security Considerations

The DATAGRAM frame shares the same security properties as the rest of
the data transmitted within a QUIC connection. All application data transmitted
with the DATAGRAM frame, like the STREAM frame, MUST be protected
either by 0-RTT or 1-RTT keys.

# IANA Considerations

This document registers a new value in the QUIC Transport Parameter Registry:

Value:

: 0x0020 (if this document is approved)

Parameter Name:

: max_datagram_frame_size

Specification:

: Indicates that the connection should enable support for unreliable DATAGRAM
frames. An endpoint that advertises this transport parameter can receive
datagrams frames from the other endpoint, up to the length in bytes provided
in the transport parameter.

This document also registers a new value in the QUIC Frame Type registry:

Value:

: 0x30 - 0x33 (if this document is approved)

Frame Name:

: DATAGRAM

Specification:

: Unreliable application data

# Acknowledgments

Thanks to Ian Swett, who inspired this proposal.
