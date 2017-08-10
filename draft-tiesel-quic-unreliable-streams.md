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
unreliable streams within QUIC-xx.
The intention of this document is to collect all unreliable streams
considerations and framing till these can be merged in [I-D.draft-ietf-quic-transport] and [I-D.draft-ietf-quic-recovery]

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
a per stream level. We consider the alternative, having a mix of
reliable and unreliable transmission within the same stream makes the
interface for applications and the too complex.


Connection Level Considerations
===============================

Unreliable Stream Support Negotiation
------------------------------------

To keep the complexity of a minimal QUIC implementation low, the
support of unreliable streams is optional. Client and server
negotiate during the TLS handshake whether the connection being set
up supports unreliable streams.



Stream Level Considerations
===========================

Stream Open
-----------

The sender opening an reliable stream must set 'R' bit of the type
byte for a STREAM frame to 1.
The sender opening an unreliable stream must set 'R' bit of the type
byte for a STREAM frame to 0.

When support of unreliable streams has not been negotiated,
receiving a STREAM frame having the 'R' bit not set is considered
an error.
An endpoint receiving such a frame MUST answer with a RST_STREAM frame
indicating a STREAM_STATE_ERROR.


Stream Close
------------

As frames of unreliable streams may not be retransmitted, the loss
of a unreliable stream frame carrying a FIN bit may lead zombie
streams. To avoid this, there are to design options:

- STREAM fames carrying the FIN bit need to be retransmitted regardless
  whether a stream is reliable or not
- Unreliable streams have to be explicitly closed with a RST_STREAM
  or CLOSE_STREAM frame indicating the final offset of the stream.
  The the RST_STREAM frame has to be resent if lost.

We advocate for the latter option, repurposing the FIN bit as 'R'
bit and changing the stream close semantic to the following:

- Once an endpoint has completed sending all stream data,
  it sends a RST_STREAM frame with error code NO_ERROR.
  The stream state becomes "half-closed (local).
- A stream in state 'open' for which a RST_STREAM frame with error
  code NO_ERROR is received, transitions to "half-closed (remote)"
  state.
  An endpoint could continue receiving frames for the stream if
  not all data advertised in 'Final Offset' was received.

This reduces the code paths that cause state transitions from open
to half-closed and eases state keeping for unreliable streams by
having reliable signaling of closing unreliable stream.
It comes with the caveat of increasing the minimal on-wire data of
a stream by 16 bytes.
Renaming RST_STREAM to CLOSE_STREAM might be useful to avoid confusion.


Stream ID 0x0
-------------

The R bit for stream ID 0x0 MUST NOT be set to 0 for obvious reasons.


Application Interface Considerations
====================================


Retransmissions within Unreliable Streams
-----------------------------------------

While unreliable streams suggest just disabling retransmissions for
these streams, applications my choose to apply arbitrary retransmission
strategies for unreliable streams, e.g. retransmit stream data
as long it will likely be delivered on-time with respect to an
application provided deadline or only retransmit certain byte ranges.


Presentation of Unreliable Streams
----------------------------------

The presentation of unreliable streams is application specific.

The anticipated use cases include:
- Data being delivered annotated with its offset as it is received.
- Data being delivered after a deadline, e.g. with an annotated list of holes and byte ranges of lost data filled with zeros.



Security Considerations {#sec}
=======================

TBD



IANA Considerations {#iana}
===================

TBD



--- back



