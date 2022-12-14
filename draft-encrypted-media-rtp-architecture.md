--
docname: draft-encrypted-media-rtp-architecture-00
title: Encrypted media over RTP architecture
category: std
ipr: trust200902
area: ART
workgroup: AVTCORE
stand_alone: yes

pi: [toc, sortrefs, symrefs]

author:
-
-
  ins:
  name:
  org:
  email:
  


normative:
  RFC2119:
  RFC3550:
  RFC3551:
  RFC3711:
  RFC4566:
  RFC7656:
  RFC8285:

informative:
  RFC6464:
  RFC6465:
  RFC6904:
  SFrame: https://sframe-wg.github.io/sframe/draft-ietf-sframe-enc.html
  WebCodecs: https://w3c.github.io/webcodecs/

--- abstract



--- middle

Introduction
============

The objective of this document is to describe how media content that is encrypted, using SFrame typically, can be sent over RTP.
It presents the overall architecture of the media pipeline and the requirements for each node of the media pipeline.
This document also provides examples of how these requirements could be implemented, as a proof of feasibility.

Media packetization and depacketization
=======================

As per {{RFC7656}}, a Media Packetizer transforms an Encoded Stream into an RTP stream.
After being sent and received over a Media Stransport, a Media Depacketizer transforms the RTP stream back into an Encoded Stream.

When a Media Encryptor is used between a Media Encoder and a Media Packetizer, the Media Encryptor tranforms the Encoded Stream
into an Encrypted Stream and the Media Packetizer (or "Encrypted Media Packetizer") transforms an Encrypted Stream into an RTP stream.
After being sent and received over a Media Stransport, an "Encrypted Media Depacketizer" transforms the RTP stream back into an Encrypted Stream.
```
                Physical Stimulus
                      |
                      V
           +----------------------+
           |     Media Capture    |
           +----------------------+
                      |
                 Raw Stream
                      V
           +----------------------+
           |     Media Source     |<- Synchronization Timing
           +----------------------+
                      |
                Source Stream
                      V
           +----------------------+
           |    Media Encoder     |
           +----------------------+
                      |
                Encoded Stream 
                      V
           +----------------------+
           |    Media Encryptor   |
           +----------------------+
                      |
                Encrypted Stream    +------------+
                      V             |            V
           +----------------------+ | +----------------------+
           |   Media Packetizer   | | | RTP-Based Redundancy |
           +----------------------+ | +----------------------+
                      |             |            |
                      +-------------+  Redundancy RTP Stream
               Source RTP Stream                 |
                      V                          V
           +----------------------+   +----------------------+
           |  RTP-Based Security  |   |  RTP-Based Security  |
           +----------------------+   +----------------------+
                      |                          |
              Secured RTP Stream   Secured Redundancy RTP Stream
                      V                          V
           +----------------------+   +----------------------+
           |   Media Transport    |   |   Media Transport    |
           +----------------------+   +----------------------+

Figure 1: Sender Side Concepts in the Media Chain with Application-level Media Encryption 
             
```
 
The packetization of Encrypted Streams does not change how the mapping between one or several Encoded Streams or Dependent Streams are mapped to the RTP Streams or how the synchronization sources(s) (SSRC) are assigned. 

An Encrypted Media Packetizer only supports Single RTP stream on a Single media Transport (SRST) when Scalable Video Coding (SVC) is in use.

The other elements on the Media Chain, like RTP-Based Redundancy, are not affected by the usage of the Encrypted Media Packetizer. 


Sending Side
=======================

The following chart focuses on inputs/output of the impacted sending side:
```
           +----------------------+
           |     Media Source     |
           +----------------------+
                      | Raw Stream:
                      | - raw content
                      | - metadata of raw content
                      V
           +----------------------+
           |    Media Encoder     |
           +----------------------+
                      |  Encoded Stream:
                      |  - encoded content
                      |  - metadata of encoded content
                      |  - metadata of raw content
                      V
           +----------------------+
           |    Media Encryptor   |
           +----------------------+
                      |  Encrypted Stream:
                      |  - encrypted content
                      |  - metadata of encoded content
                      |  - metadata of raw content
                      V
           +--------------------+
           |   Media Packetizer |
           +--------------------+
                      |  RTP Stream:
                      |  - RTP payloads (from encrypted content)
                      |  - RTP header extensions (from metadata)
                      V
                     ... 

Figure 2: A closer look at impact of a Media Encryptor on the sending side.
```

Media Encryptor
-----------------------

The Media Encoder takes a Raw Stream consisting of raw content and metadata of raw content and generates an Encoded Frame consisting of encoded content (typically an array of bytes), and metadata of encoded content.  The metadata of encoded content describes the encoded content.  {{WebCodecs}} represents, encoded content in units of EncodedAudioChunk and EncodedVideoChunk, and metadata of encoded content in units of EncodedAudioChunkMetadata and EncodedVideoChunkMetadata.

The Media Encryptor takes an Encode Stream and metadata of raw content and generates an Encrypted Stream consisting of encrypted content. It also separates out the metadata that will be needed by processing units that do not have encryption keys (and thus can work only with an Encrypted Stream and not with an Encoded Stream).   Metadata needed by such processing units is passed directly to the Media Packetizer, but metadata that should not be available by such processing units is not passed directly to the Media Packetizer (but may be included in the encrypted content).  {{SFrame}} is an example of encrypted content.

It is the responsibility of the application using the Media Encryptor to provide its encoded content in meaningful units of encoded content.
In the common case of a video codec, the unit of encoded content is a video frame in byte format (h264 annex b for example).

If a unit of encoded content smaller than a video frame is useful for a particular application, it is the responsibility of the Media Encoder to provide the Media Encryptor such units. Each unit of encoded content will be encrypted and sent to the Encrypted Media Packetizer separately, as a unit of encrypted content. The typical case is a video codec supporting spatial scalability: each spatial layer will typically be split in its own encoded content unit.

RTP packetization
-----------------------

The Encrypted Media Packetizer takes an Encrypted Stream and metadata and generates an RTP stream.
It will typically put the encrypted content into RTP payloads and a subset of metadata in RTP headers such that the RTP packets do not exceed the network maximum transmission unit.
By design, the Encrypted Media Packetizer is not expected to understand the format of the encrypted content, and treats it like an opaque array of bytes.  
It must not modify the encrypted content.
The RTP packets MUST have the necessary information to reconstruct the encrypted content and payload type of the encoded content without decrypting the encrypted content.  This prevents needing to negotiate many RTP payload types; a single payload type can represent encrypted content of a particular type (for example, {{SFrame}}, rather than one per type of encoded content). 

For video, it may be particularly useful to include metadata like whether the frame is a key frame or the spatial layer of the frame if spatial scalability is used.  This information can, for instance, be sent using a Dependency Descriptor header extension.  Other information like the marker bit or RTP timestamp are also set based on the metadata.

Receiving side
=======================

The following chart focuses on inputs/output on the receiving side:
```
               Source RTP Stream
                      |
                      V
           +----------------------+
           |  Media Depacketizer  |
           +----------------------+
                      |  encrypted content
                      |  metadata of raw content
                      |  metadata of encoded content
                      V
           +----------------------+
           |    Media Decryptor   |
           +----------------------+
                      |  encoded content
                      |  metadata of raw content
                      |  metadata of encoded content
                      V
           +----------------------+
           |   Media Decoder      |
           +----------------------+
                      |  raw content
                      |  metadata of raw content
                      V
                     ... 

Figure 3: A closer look at the impact of a Media Decryptor on the receiving side.
```

On the receiving side, the Encrypted Media Depacketizer reconstructs the encypted content, metadata, and payload type of the encoded content from RTP packets.  
For video content, this can be based on reading the Dependency Descriptor header extension if present.  
The Media Decryptor decrypts the decoded content into encoded content.
The correct Media Decoder is identified by the payload type of the encoded content and the encoded content is passed to the Media Decoder, which produces raw content along with metadata of raw content.

Intermediaries that do not have encryption keys (such as SFUs) can also reconstruct the encrypted content, metadata, and payload type of the encoded content, but cannot decrypt the encrypted content.  This is particularly useful for servers (such as SFUs) that need to have a basic understanding of each frame they receive so as to decide to forward it or not and to which endpoint. Such processing units may also repacketize the encrypted content and metadata into other RTP streams. Such intermediaries must not modify the encrypted content.



Transmitted metadata
-----------------------

The metadata available to intermediaries without encryption keys should be limited to what is strictly necessary.

For audio, such metadata might include:
- Opus TOC
For video, the following configuration information might include:
- Stream configuration information: resolution, quality, frame rate...
- Codec specific configuration information: codec profile like profile_idc...
- Frame specific information: whether the stream is decodable when starting from this frame, whether the frame is skippable...


Proof of concept
=======================

RTP packet generation and reception
-----------------------

One possible implementation may meet these requirements as follows:
1. The payload type of the encrypted content is put into the RTP header.
1. The payload type of the encoded content is the first byte of the RTP payload.
1. The remaining RTP payload is a chunk of the encoded content broken up such that the concatenation of RTP payloads (after removing the payload type prefix) in order of sequence number is equal to the encoded content.
1. Send SVC encrypted content using one SSRC in ascending layer order, with the same RTP timestamp, and only the last RTP packet of the last spatial layer frame will have the marker bit set to 1.
1. Send SVC metadata using the Dependency Descriptor header extension. The first RTP packet of the frame will have its start_of_frame equal to 1 and the last packet will have its end_of_frame equal to 1.  Such information may be procided by WebCodecs SvcOutputMetadata
(see https://docs.google.com/presentation/d/1lFAUSvApbBYfBNJH_xcRW0YjD0aF5T1ZqjyyDJMesJw/edit#slide=id.g1a4ac56601a_3_7).


This makes it easy to identify that the RTP payload is encrypted content encoded in a particular format, and to know both the encrypted content and the payload type of the encoded content.  It also makes it easy for SFUs to know SVC metadata it needs to forward media correctly.


SDP negotiation
-----------------------

One possible way to do SDP negotiation is as follows:
1.  The payload types of encoded content, RTX, and FEC are neogiated like normal.
1.  Payload types for encrypted content are also negotiated, one for each type of encoded content.


An example:
```
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=setup:actpass
a=mid:1
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=extmap:4 https://aomediacodec.github.io/av1-rtp-spec/#dependency-descriptor-rtp-header-extension
a=sendrecv
a=rtpmap:96 vp9/90000
a=rtpmap:97 vp8/90000
a=rtpmap:98 encrypted/90000
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=96
a=rtpmap:100 rtx/90000
a=fmtp:100 apt=97
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=98
```

Q: What frequency should we use for audio? One per the different frequencies of each audio codec?

IANA Considerations
-----------------------

Two new media subtypes could be registered with IANA, as described in this section.
This registration could be done using the registration template {{REF}} and following [[RFC3555]].

## Registration of audio/encrypted

   Type name: audio

   Subtype name: encrypted

   Required parameters: none

   Optional parameters: none

   Encoding considerations: This format is framed (see Section 4.8 in the template document [3]) and contains binary data.

   Security considerations: TBD.

   Interoperability considerations: TBD

   Published specification: TBD.

   Applications that use this media type: TBD.
   
   Additional information: none

   Intended usage: COMMON

   Restrictions on usage: TBD

   Author:

   Change controller:
      
# Registration of video/encrypted

   Type name: video

   Subtype name: encrypted

   Required parameters: none

   Optional parameters: none

   Encoding considerations: This format is framed (see Section 4.8 in the template document [3]) and contains binary data.

   Security considerations: TBD.

   Interoperability considerations: TBD

   Published specification: TBD.

   Applications that use this media type: TBD.
   
   Additional information: none

   Intended usage: COMMON

   Restrictions on usage: TBD

   Author:

   Change controller:


Security Considerations
=======================

This document describes an architecture and requirements for encrypting media content.
It does not introduce any new security considerations beyond those already well documented in the RTP protocol.

--- back
