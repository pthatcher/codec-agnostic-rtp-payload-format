---
docname: draft-encrypted-media-rtp-payload-easy-repacketization
title: "RTP payload format for encrypted media"
category: std
date: {DATE}

ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Audio/Video Transport Core Maintenance"
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]
submissiontype: IETF

author:
 -
    ins: P. Thatcher
    name: Peter Thatcher
    organization: Microsoft
    email: pthatcher@microsoft.com

--- abstract

This document specifies how to packetize RTP for encrypted media as described in draft-encrypted-media-rtp-architecture.

--- middle

# Introduction

The packetization format described in this doc is designed to make it easy for intermediaries such as SFUs to repacketize between different sizes of RTP payload, such as when repacketizing between RTP over UDP and RTP over QUIC even when packets arrive out of order.

# Terminology and Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Format

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|E| MPT         |FO | Frame ID                                  |        
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset                        |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
|                          Chunk                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

E bit: if 0, nothing is after the Payload Type; if 1, the FO bits follows the Payload Type.

MPT: the payload type of the encoded content (different than the payload type of encrypted content)

FO bits: if 11, the Frame ID is 22 bits long and the Offset is 24 bits long;
         if 10, the Frame ID is 22 bits long and the Offset is 0 bits long;
         if 01, the Frame ID is 0 bits long and the Offset is 22 bits long;
         if 00, the Frame ID is 22 bits long and the Offset is 16 bits long;
          
Chunk: a chunk of the encrypted content, starting at the Offset and ending a the Offset plus the length of the Chunk.

The Packetizer MUST generate RTP packets from an encrypted frame such that a Depacketizer may generate the content of the encrypted frame by doing the following:
1.  Let F be the set of RTP packets that share a Frame ID.
1.  Let L be the packet in F with the largest Offset.
1.  Let B a byte buffer B of size equal to the Offset of L plus the length of the Chunk of L
1.  For each RTP packet P in F, write the Chunk of P to B starting at the offset of P.


If the Frame ID is not present, then the Packetizer MUST put a frame ID in an RTP header extension such as the Dependency Descriptor or ensure it may be inferred from other information, such as the RTP marker bit or the Payload Type of the encoded content (although such infererence may require additional buffering and delay).

If the Offset is not present, then it may be inferred from RTP packets with sequential sequence numbers (although such infererence may require additional buffering and delay).

# Format for when the Payload Type of the encoded content that can be inferred

A second version of this format may be used which excludes the payload type for cases where
the Payload Type of the encoded content may be inferred from the rest of the RTP packet:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|FO | Frame ID                                  | Offset ...       
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 ... Offset       |                                             |
+-+-+-+-+-+-+-+-+-+                                             |
|                           Chunk                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

This version requires that either a Frame ID or Offset be present.

# Format for when everything can be inferred

A third version of this format may be used which excludes everything: the Payload Type, Frame ID, and Offset.  This is only useful in the case where everything can inferred.  In this version, the Packetizer simply divides the encoded frame into sequential Chunks (if there is only one, the Chunk is exactly equal to the content of the encrypted frame).

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Chunk                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

# Examples

## Audio when there is only one audio Payload Type

For audio, the content of the encoded frame fits into one RTP packet.  If there is only one audio Payload Type (such as Opus), then the Packetizer may use the 3rd version of this format in which case it simply sticks the content of the encoded frame into the RTP payload.

## Audio when there are many audio Payload Types

If the Payload Type cannot be inferred, the Packetizer may choose to use the 1st version of this format and include the MPT.  Because the Frame ID and Offset can be inferred trivially, the Packetizer may set the E bit to 0.

## Video with Dependency Descriptor or other header extension containing a Frame ID

If the Packetizer is already including the Dependency Descriptor or similar RTP header extension
that has a Frame ID or equivalent value that allows correlating RTP packets to encrypted frames
and the ordering of encrypted frames, the Packetizer use the 1st version of this format and set the E bit to 1 and the FO bits to 01 and include the Offset in each RTP packet.  This would make it easy for an SFU to repacketize at the cost of a small amount of packet overhead.

The Packetizer may also choose to reduce packet overhead at the cost of more difficulty in SFU repacketization by setting the E bit to 1 and excluding the Offset from the RTP packets.

If the Payload Type can be inferred from the Dependency Descriptor or other information in the RTP packet, the Packetizer may choose to use the 2nd version of this format.

## Video without Dependency Descriptor or other header extension containing a Frame ID

If the Packetizer is not using a header extension such as the Dependency Descriptor, it may set the E bit to 1 and the FO bits to 11 to include both the Frame ID and Offset.  This would make it easy for an SFU to repacketize at the cost of packet overhead.  If the Offset fits within 16 bits, the Packetizer may set the FO bits to 00.

The Packetizer may also choose to reduce packet overhead at the cost of more difficulty in SFU repacketization by setting the E bit to 1 and the FO bits to either 01 or 10 and exclude either the Frame ID or Offset.  Of it may exclude both by setting the FO bits to 00.

# Negotiation

When negotiating RTP payload types, an RTP payload type must be negotiated for each version of this format the endpoints wish to use.  The 3 versions of the format are named "encrypted", "generic", and "simple", respectively.