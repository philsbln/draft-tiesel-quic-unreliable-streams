---
title: Considerations for Unreliable Streams in QUIC
abbrev: QUIC Unreliable Streams
docname: draft-tiesel-quic-unreliable-streams-latest
date: 2017-10-24
category: info

ipr: trust200902
area: Transport
workgroup: QUIC Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: P. S. Tiesel
    name: Philipp S. Tiesel
    organization: TU Berlin
    street: Marchstr. 23
    city: Berlin
    country: Germany
    email: philipp@inet.tu-berlin.de
 -
    ins: M. Palmer
    name: Mirko Palmer
    organization: TU Berlin
    street: Marchstr. 23
    city: Berlin
    country: Germany
    email: mirko@inet.tu-berlin.de
 -
    ins: B. Chandrasekaran
    name: Balakrishnan Chandrasekaran
    organization: TU Berlin
    street: Marchstr. 23
    city: Berlin
    country: Germany
    email: balac@inet.tu-berlin.de
 -
    ins: A. Feldmann
    name: Anja Feldmann
    organization: TU Berlin
    street: Marchstr. 23
    city: Berlin
    country: Germany
    email: anja@inet.tu-berlin.de
 -
    ins: J. Ott
    name: Joerg Ott
    organization: TU Munich
    street: Boltzmannstraße 3
    city: Garching bei München
    country: Germany
    email : ott@in.tum.de

normative:

informative:
  RFC2119:
  I-D.draft-ietf-quic-transport-07:
  I-D.draft-ietf-quic-recovery-06:
  I-D.draft-ietf-quic-applicability-00:



--- abstract

This memo outlines unreliable stream support for QUIC.
It contains a collection of considerations and requirements for unreliable streams in QUIC.
It provides three alternatives demonstrating how unreliable streams can be realized.
The intention of this document is to collect considerations regarding
unreliable streams in QUIC and to frame the design space and give an example how unreliable stream support could be realized.


--- middle



Conventions and Definitions
===========================

The words "MUST", "MUST NOT", "SHALL", "SHALL NOT", "SHOULD", and
"MAY" are used in this document. It's not shouting; when these
words are capitalized, they have a special meaning as defined
in {{RFC2119}}.


Introduction        {#intro}
============

There are many use cases for unreliable delivery of stream data, e.g., to meet deadlines for data delivery in the presence of short time congestion by avoiding head of line blocking.
Still, some control data or metadata often needs to be transmitted reliably.

This memo describes how QUIC can provide reliable and unreliable transmission within the same connection.
This model allows applications to request reliable or unreliable transmission in QUIC on a per stream level.
For an unreliable stream, the QUIC implementation can decide which frames to retransmit.
Thus, it is possible to implement a mix of reliable and unreliable transmission within the same stream.

This draft is based on [I-D.draft-ietf-quic-transport-07] with the addition of uni-directional streams (https://github.com/quicwg/base-drafts/pull/872) and variable length integer encoding (https://github.com/quicwg/base-drafts/pull/877)
All content of this memo is supposed to be merged into
[I-D.draft-ietf-quic-transport-07], [I-D.draft-ietf-quic-recovery-06] and,
[I-D.draft-ietf-quic-applicability-00] once unreliable streams get
integrated with QUIC.




Connection Level Considerations
===============================

Unreliable Stream Support Negotiation
------------------------------------

Support of unreliable streams should be optional to reduce the complexity of a minimal QUIC implementation.
An endpoint signals its willingness to receiving unreliable stream
frames during the TLS handshake using the transport parameter
allow_unreliable_streams in analogy to the omit_connection_id flag specified in [I-D.draft-ietf-quic-transport-07].
An application that makes no use of partial delivery of stream data, should not signal willingness to allow unreliable streams.



Stream Level Considerations
===========================




Stream Open
-----------

For each stream, the receiver must be able to know whether he can rely on the sender retransmitting lost stream data.
Therefore, an endpoint opening an unreliable stream MUST indicate the fact that for a particular stream lost stream data might not be retransmitted.
The indication must either be carried on each frame that could be received first by an endpoint or it must be transmitted reliable and acknowledged before the first frame of an unreliable stream is sent.

We anticipate two options how to indicate whether a stream is unreliable:

 - Indicate unreliable streams in the stream frame header.
 - Leave the signaling of unreliable streams completely to the application layer.

The authors advocate for explicitly signaling unreliable streams in
the stream frame header.
This approach does not introduce additional interwinding between QUIC and the application.
It also reduces the complexity of applications where only some streams should be transmitted unreliably.
See {{str_ind}} for a proposal how to change the stream frame header when a an endpoint has signaled the willingness to allow unreliable streams.


Stream Close {#stream_close}
------------

As frames of unreliable streams may not be retransmitted, the loss of a frame indicating the end of a stream may introduce a zombie stream state.
A QUIC version with unreliable stream support MUST make sure that either such a zombie state does not occur or include a mechanism to clean up zombie streams, e.g. by using a window of active unreliable stream ids.

The authors advocate for solving this issue by reliable transmitting the final offset for all stream that consist of more than a single stream frame.
The retransmit of the stream end can be achieved by sending an empty stream frame with the final offset and the FIN bit sent or by using a RST_STREAM frame.

The last requirement can be relaxed for unidirectional streams that only consist of a single stream frame – See {{stream_msg}} for details.


Stream as a message {#stream_msg}
-------------------

Unreliable streams are particularly useful if QUIC streams are used as a message abstraction.
If the messages can be sent using a single stream frame on an unidirectional stream, the state committed at the sender- and receiver side can be minimized and signaling of stream close can be omitted.
The loss of such a frame does not introduce state at the perceived receiver, nor does it require the sender to keep state for retransmission.


Stream ID 0x0
-------------

Data of stream 0x0 MUST be transmitted reliably as TLS expects
reliable transmission.


Congestion Control on Unreliable Streams
----------------------------------------

Unreliable streams are subject to regular congestion control.

It should be clarified that ACK and RST_STREAM Frames are not subject to congestion control.


Flow Control on Unreliable Streams
----------------------------------

Unreliable streams are subject to regular flow control on connection and stream level.



Application Interface Considerations
====================================

Retransmissions within Unreliable Streams
-----------------------------------------

While unreliable streams suggest just disabling retransmissions for
these streams, applications my choose to apply arbitrary retransmission
strategies for unreliable streams, e.g., retransmit stream data
as long it will likely be delivered on-time with respect to an
application provided deadline or only retransmit certain byte ranges.

A QUIC implementation that implements retransmissions on a per-packet
basis, therefore, may retransmit unreliable stream data even if not
requested by the application.


Presentation of Unreliable Streams
----------------------------------

The presentation of unreliable streams is application specific.

The anticipated use cases include:

 - Data being delivered annotated with its offset as it is received.
 - Data being delivered after a deadline, e.g., with an annotated list of holes and byte ranges of lost data filled with zeros.
 - Data being delivered as concatenation of the stream fragments received.
   This can be useful if the application's framing is aligned with the QUIC stream frames.


Prioritization of Unreliable Streams
------------------------------------

Prioritization of Unreliable streams uses the same priority mechanism as reliable streams.

Applications that want UDP-like behavior and thus wand to avoid data sent over unreliable streams queued in send buffers  must make sure that:

 - The flow control windows are large enough to send at any given point in time
 - The data rate sent in all frames stays below the one permitted by the congestion window.
 - The priority of unreliable streams is high enough to transmit data, even if there are retransmissions outstanding on other streams.

To enable writing portable applications, guidelines how prioritization should be handled in a QUIC implementation and how it is exposed to the application are required.



Implemenation Proposal {#str_ind}
======================

This proposal realizes unreliable streams by modifying
[I-D.draft-ietf-quic-transport-07] with the addition of uni-directional streams (https://github.com/quicwg/base-drafts/pull/872) and variable length integer encoding (https://github.com/quicwg/base-drafts/pull/877) to support unreliable streams.
It modifies the Stream ID to signal whether a stream is reliable in analogy to the way unidirectional streams are signaled.
In addition, it addresses the Stream Close concerns raised in {{stream_close}}.

Stream Identifiers {#stream-id}
------------------

Streams are identified by an variable-length integer, referred to as the Stream ID.
The least significant three bits of the Stream ID are used to identify the type of stream (unidirectional or bidirectional),  (unreliable or reliable) and the initiator of the stream.

The second least significant bit (0x2) of the Stream ID differentiates between unidirectional streams and bidirectional streams.
Unidirectional streams always have this bit set to 1 and bidirectional streams have this bit set to 0.

The third least significant bit (0x4) of the Stream ID differentiates between unreliable and reliable streams.
Unreliable streams always have this bit set to 1 and reliable streams have this bit set to 0.

The three type bits from a Stream ID therefore identify streams as summarized in {{stream-id-types}}.

| Low Bits | Stream Type                                  |
|:---------|:---------------------------------------------|
| 0x0      | Client-Initiated, Bidirectional , Reliable   |
| 0x1      | Server-Initiated, Bidirectional , Reliable   |
| 0x2      | Client-Initiated, Unidirectional, Reliable   |
| 0x3      | Server-Initiated, Unidirectional, Reliable   |
| 0x4      | Client-Initiated, Bidirectional , Unreliable |
| 0x5      | Server-Initiated, Bidirectional , Unreliable |
| 0x6      | Client-Initiated, Unidirectional, Unreliable |
| 0x7      | Server-Initiated, Unidirectional, Unreliable |

{: #stream-id-types title="Stream ID Types"}


Stream Close
------------

Streams MUST be explicitly closed. This can either be done by setting the FIN bit on the frame carrying the final offset of the stream or by using a RST_STREAM frame indicating the final offset.

For unreliable stream transmitted using more than one stream frame,  the stream close has to be transmitted reliably to prevent zombie stream state.
If the package containing the FIN flagged stream frame is lost, and the data in this stream is not to be retransmitted, the sender can (re-)transmit the stream close by sending an empty FIN flagged stream frame carrying the final offset.



Security Considerations {#sec}
=======================

Unreliable Streams open a few additional risks of information disclosure.

 - An active, on path attacker can drop selected frames and see whether the transmission timings change to see whether unreliable streams are used.
 - Using streams as a message as described in {#stream_msg} will most likely result in a PDUs that resemble the packet size of the messages.
   If not mitigated, this can disclose individual message length.


IANA Considerations {#iana}
===================

TBD



Acknowledgements
================

This work has been supported by Leibniz Prize project funds of DFG - German Research Foundation: Gottfried Wilhelm Leibniz-Preis 2011 (FKZ FE 570/4-1) and received funding from the European Union’s Horizon 2020 research and innovation programme 2014-2018 under grant agreement No. 644866, Scalable and Secure Infrastructures for Cloud Operations (SSICLOPS).


--- back


