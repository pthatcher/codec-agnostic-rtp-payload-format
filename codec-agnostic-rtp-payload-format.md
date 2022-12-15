--
docname: draft-encrypted-media-rtp-payload-format
title: RTP payload format for encrypted media
category: std
ipr: trust200902
area: ART
workgroup: AVTCORE
stand_alone: yes

pi: [toc, sortrefs, symrefs]

author:
-
  ins: S. Garcia Murillo
  name: Sergio Garcia Murillo
  org: CoSMo
  email: sergio.garcia.murillo@cosmosoftware.io

-
  ins: Y. Fablet
  name: Youenn Fablet
  org: Apple Inc.
  email: youenn@apple.com 
  
-
  ins: A. Gouaillard
  name: Alex Gouaillard
  org: CoSMo
  email: alex.gouaillard@cosmosoftware.io



normative:
  RFC2119:
  RFC3550:
  RFC3551_
  RFC3711:
  RFC4566:
  RFC7656:
  RFC8285:

informative:
  RFC6464:
  RFC6465:
  RFC6904:
  SFrame:

--- abstract



--- middle

Introduction
============

The objective of this spec is to create an RTP packetization format suitable for encrypted media that allows SFUs to perform routing and layer selection without decrypting the payloads.  The architecture and requirements of such a format is described in draft-encrypted-media-rtp-architecture.
 
RTP packetization
=======================

Encrypted frames are broken up into RTP packets with sequential sequence numbers such that the concatenation of RTP payloads is equal to the encrypted frames.  Deliniation between encrypted frames is accomplished using the RTP marker bit.  The marker bit of each RTP packet in a frame MUST be set according to the audio and video profiles specified in [[rfc3551]].  When using SVC with a single SSRC, the spatial layer frames are sent in ascending order, with the same RTP timestamp, and only the last RTP packet of the last spatial layer frame will have the marker bit set to 1.


Payload Type Header Extension
-----------------------------
The encoded content payload type (the payload type before encryption) is sent in a header extension with the URI "urn:ietf:params:rtp-hdrext:associated-payload-type", defined as follows:


```
                    0                   1
                    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |  ID   | len=0 |R|     PT      |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
Figure 1: The One-Byte Header Format

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |      ID       |     len=1     |R|     PT      |    0 (pad)    |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
Figure 2: The Two-Byte Header Format

The R bit is reserved for future use.

--- back
