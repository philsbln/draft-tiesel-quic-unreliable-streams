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


Security Considerations {#sec}
=======================


IANA Considerations {#iana}
===================

None



--- back



