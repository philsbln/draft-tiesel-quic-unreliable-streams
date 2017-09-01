---
title: Considerations for Unreliable Streams in QUIC
abbrev: QUIC Unreliable Streams
docname: draft-tiesel-quic-unreliable-streams-latest
date: 2017-06-27
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
    organization: Berlin Institute of Technology
    street: Marchstr. 23
    city: Berlin
    country: Germany
    email: philipp@inet.tu-berlin.de
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
  I-D.draft-ietf-quic-transport-04:
  I-D.draft-ietf-quic-recovery-04:
  I-D.draft-ietf-quic-applicability-00:



--- abstract

This memo outlines support for unreliable streams in QUIC.
This draft contains a collection of considerations and requirements
for unreliable streams in QUIC as well as a proposal how to implement
unreliable streams within quic-draft-05.
The intention of this document is to collect all unreliable streams
considerations and framing till these can be merged in [I-D.draft-ietf-quic-transport], [I-D.draft-ietf-quic-recovery] and, [I-D.draft-ietf-quic-applicability].


--- middle

Conventions and Definitions
===========================

The words "MUST", "MUST NOT", "SHALL", "SHALL NOT", "SHOULD", and
"MAY" are used in this document. It's not shouting; when these
words are capitalized, they have a special meaning as defined
in {{RFC2119}}.



Introduction        {#intro}
============

There are many use cases for use cases for unreliable streams,
especially in cases where an application has to meet deadlines for
data delivery and have to avoid head of line blocking.
But for many of these use cases, there is still a need to transmit
some data, often control or meta data, reliably.

This memo describes how QUIC can provide reliable and unreliable
transmission within the same connection. We advocate to allow the
application to request reliable or unreliable transmission in QUIC on
a per stream level. As we do allow a QUIC implementation to retransmit
unreliable stream frames, it is possible to implement a mix of reliable
and unreliable transmission within the same, tagged unreliable, stream.



Connection Level Considerations
===============================

Unreliable Stream Support Negotiation
------------------------------------

To keep the complexity of a minimal QUIC implementation low, the
support of unreliable streams is optional.
An endpoint signals its willingness to receiving unreliable stream
frames during the TLS handshake using the transport parameter
accept_unreliable_stream_frames (value TBD – used as flag the same
way as omit_connection_id specified in [I-D.draft-ietf-quic-transport]).

Connections on behalf of applications that make no use of partial
delivery of stream data should not send accept_unreliable_stream_frames
even if the QUIC implementation support it.



Stream Level Considerations
===========================

Stream Open
-----------

In addition to the stream open specified in [I-D.draft-ietf-quic-transport], an endpoint opening a stream
MUST indicate whether the stream is reliable and therefore the
receiver can rely on the sender retransmitting lost stream data.

We see two options how to indicate whether a stream is reliable or not:

 - Leave it completely to the application layer.
 - Indicate it in the stream frame header by repurposing the
   'F' (FIN) bit as 'R' (RELIABLE) bit and indicate the stream close
   using CLOSE_STREAM / RST_STREAM frames.

See {{app_ind}} for a proposal specifying the former and
{{str_ind}} for a proposal specifying the latter.

The authors advocate for explicitly signaling unreliable streams in
the stream frame header as it does not introduce additional
interwinding between QUIC and the application and makes usage of
unreliable streams for a limited amount of stream data much easier.


Stream Close
------------

As frames of unreliable streams may not be retransmitted, the loss
of a frame indicating the end of a stream may introduce zombie streams.

A final protocol MUST make sure that either such a zombie state does
not occur (as the proposals in {{app_ind}} and {{str_ind}} do) or
include a mechanism to clean up zombie streams, e.g. by using a
window of active unreliable stream ids.


Stream ID 0x0
-------------

Data of stream 0x0 MUST be transmitted reliably as TLS expects
reliable transmission.


Congestion Control on Unreliable Streams
----------------------------------------

Unreliable streams are subject to regular congestion control.
CLOSE_STREAM Frames are, like ACK and RST_STREAM frames not subject to congestion control.


Flow Control on Unreliable Streams
----------------------------------

Unreliable streams are subject to regular flow control on connection and stream level.



Application Interface Considerations
====================================

Retransmissions within Unreliable Streams
-----------------------------------------

While unreliable streams suggest just disabling retransmissions for
these streams, applications my choose to apply arbitrary retransmission
strategies for unreliable streams, e.g. retransmit stream data
as long it will likely be delivered on-time with respect to an
application provided deadline or only retransmit certain byte ranges.

An QUIC implementation that implements retransmissions on a per-packet
basis therefore may retransmit unreliable stream data even if not
requested by the application.


Presentation of Unreliable Streams
----------------------------------

The presentation of unreliable streams is application specific.

The anticipated use cases include:

 - Data being delivered annotated with its offset as it is received.
 - Data being delivered after a deadline, e.g. with an annotated list of holes and byte ranges of lost data filled with zeros.


Prioritization of Unreliable Streams
------------------------------------

Unreliable streams are not prioritized in any special way.

Applications that need UDP like behavior must make sure that:

 - The flow control windows are large enough to send at any given point in time
 - The data rate sent in all frames stays below the one permitted by the congestion window.
 - The priority of unreliable streams is high enough to transmit data, even if there are retransmissions outstanding on other streams.

To enable writing portable applications, guidelines how prioritization should be
handled in a QUIC implementation and how it is exposed to the application are required.


Security Considerations {#sec}
=======================

TBD



IANA Considerations {#iana}
===================

TBD




--- back

Proposal with application layer indicated relniabliliy {#app_ind}
===========================

This implementation proposal lets the decision and indication of
unreliable transmission completely to the application.
In this proposal, the state space and error handling for applications
can become quite complex when control messages at the start of a stream
are transmitted unreliably.
The advantage of this implementation proposal requiring only minimal
changes to [I-D.draft-ietf-quic-transport] and [I-D.draft-ietf-quic-recovery].

Stream Close
------------

As frames of unreliable streams may not be retransmitted, the loss
of a unreliable stream frame carrying a FIN bit may lead to zombie
streams.
To prevent this this STREAM fames carrying the FIN bit MUST to
be retransmitted if lost regardless whether a stream is reliable or not.



Proposal with stream frame indicated relniabliliy {#str_ind}
===========================

This implementation proposal repurposes the 'F' (FIN) bit
from  [I-D.draft-ietf-quic-transport] as 'R'(RELIABLE) bit and
indicates the stream close using CLOSE_STREAM / RST_STREAM frames.

Stream Open
-----------

 - The sender opening an reliable stream must set 'R' bit of the type
byte for a STREAM frame to 1.
 - The sender opening an unreliable stream must set 'R' bit of the type
byte for a STREAM frame to 0.

A client that has not indicated its willingness to receive
unreliable stream frames as using the transport parameter
accept_unreliable_stream_frames MUST answer with a RST_STREAM frame
indicating a STREAM_STATE_ERROR when receiving a STREAM frame having
the 'R' bit not set.

All frames of a stream MUST have the R bit to the same value.


Stream Close
------------

Streams MUST explicitly closed with a CLOSE_STREAM frame indicating the stream ID and final offset of the stream to prevent zombie streams.
(Alternative implementation option: RST_STREAM frame with error code NO_ERROR indicating the final offset).

The the CLOSE_STREAM frame has to be resent if lost
even if the stream frames of this stream are transmitted unreliably.

 - Once an endpoint has completed sending all stream data,
   it sends a CLOSE_STREAM frame.
   The stream state becomes "half-closed (local).
 - A stream in state 'open' for which a CLOSE_STREAM frame is received,
   transitions to "half-closed (remote)" state.
   An endpoint could continue receiving frames for the stream if
   not all data advertised in 'Final Offset' was received.

This reduces the code paths that cause state transitions from open
to half-closed and eases state keeping for unreliable streams by
having reliable signaling of closing unreliable stream.
It comes with the caveat of increasing the minimal on-wire data of
a stream by at least four bytes.



CLOSE_STREAM Frame
------------------

The CLOSE_STREAM frame indicates the final offset of a stream
and therefore replaces the F bit.

I uses a type value between 0x80 and 0x9f.
The type byte for a CLOSE_STREAM frame contains embedded flags,
and is formatted as "100RSSOO".  These bits are parsed as follows:

 - The R bit is set to 1 for complete reliable streams and o 0 otherwise.

 - The "SS" bits encode the length of the Stream ID header field.
   The values 00, 01, 02, and 03 indicate lengths of 8, 16, 24, and
   32 bits long respectively.

 - The "OO" bits encode the length of the Offset header field.  The
   values 00, 01, 02, and 03 indicate lengths of 0, 16, 32, and 64
   bits long respectively.

A CLOSE_STREAM frame is shown below.

~~~~
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Stream ID (8/16/24/32)                   ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                 Final Offset (0/16/32/64)                   ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
