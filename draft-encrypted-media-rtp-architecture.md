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

As per {{RFC7656}}, the encrypted media packetizer will define a Media Packetizer that transforms a single encrypted encoded frame into one or several RTP packets.
These RTP packets are sent over the wire to a Media Depacketizer that will reconstruct the encrypted encoded frame.
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
 
This encrypted media packetization does not change how the mapping between one or several encoded or dependent streams are mapped to the RTP streams or how the synchronization sources(s) (SSRC) are assigned. 

The encrypted media packetizer only supports Single RTP stream on a Single media Transport (SRST) when Scalable Video Coding (SVC) is in use.

The other elements on the Media Chain, like RTP-Based Redundancy, are not affected by the usage of the encrypted media packetizer. 


Sending Side
=======================

The following chart focuses on inputs/output of the impacted sending side:
```
           +----------------------+
           |     Media Source     |
           +----------------------+
                      | media frame metadata
                      | media frame content
                      V
           +----------------------+
           |    Media Encoder     |
           +----------------------+
                      |  media frame metadata
                      |  encoded media frame metadata
                      |  encoded media frame content
                      V
           +----------------------+
           |    Media Encryptor   |
           +----------------------+
                      |  media frame metadata
                      |  encoded media frame metadata
                      |  encrypted media frame content
                      V
           +----------------------+
           |   Media Packetizer   |
           +----------------------+
                      |  RTP packets:
                      |  - payloads generated from encrypted content
                      |  - header extensions generated from
                      |    media frame metadata and encoded media frame metadata
                      V
                     ... 

Figure 2: A closer look at media content encryption impact on the sending side.
```

Media Encryptor
-----------------------

The media encryptor is placed directly after the media encoder.
The main purpose of the media encryptor is to encrypt the media frame encoded content.
The media encryptor will also filter out metadata that is not useful for the processing units that will process the encrypted content without having the encryption keys.

The media encoder takes as input a media frame consisting of media frame content and media frame metadata.
It generates an encoded frame that consists of encoded frame content, an array of bytes typically, and encoded frame metadata.
The encoded frame metadata describes the encoded frame content and is typically represented in {{WebCodecs}} as a EncodedAudioChunk or EncodedVideoChunk,
complemented by EncodedAudioChunkMetadata or EncodedVideoChunkMetadata.

The media encryptor generates an encrypted encoded frame using a particular format like {{SFrame}} from the encoded frame.
This encrypted encoded frame consists of encrypted encoded content and the same encoded frame metadata of the encoded frame.

The encrypted encoded frame is then sent to the encrypted media packetizer with the media frame metadata.

It is the responsibility of the application using the media encryptor to group media content in meaningful frames.
In the common case of a video codec, the encoded frame is the frame in byte format (h264 annex b for example) generated by the encoder. 

In general, if granularity below the video frame is useful for a particular application, it is the responsibility of the media encoder to provide
to the media encryptor the smallest processable media content unit as an individual encoded frame.
Each encoded frame will then be encrypted and sent to the encrypted media packetizer separately.
The typical case is a video codec supporting spatial scalability: each spatial layer will typically be split in its own encoded frame.

RTP packetization
-----------------------

The packetizer is responsible to transform each encrypted media frame into an array of RTP packets.
At a high level, the encrypted media packetizer will put the media frame encrypted content within RTP payloads.
It will also transmit a subset of the metadata provided by the media encryptor, typically in RTP headers.

When the encrypted media packetizer receives an encrypted media frame from the application, it fragments the content to be transmitted in multiple RTP packets to ensure packets do not exceed the network maximum transmission unit.
Each RTP packet has the necessary information to retrieve the payload type associated with the media encoder without decrypting the encrypted content.
This allows reducing the number of payload type codes on the SDP exchange, a single payload type code for the encrypted media packetization can be used for each media type.

The encrypted media packetizer, by design, is not expected to understand the format of the encrypted media content to transmit.
The content to be transmitted is treated as a binary blob by the packetizer, so the decision about the boundaries of each fragment is decided arbitrarily by the packetizer.
The packetizer or any relaying server MUST NOT modify the encrypted content.
It must be feasible from the whole set of RTP packets corresponding to an encrypted media frame to exactly reproduce the binary content of the origina frame encrypted content.

The packetizer is also responsible to set the RTP header information appropriately.
The packetizer will generate the RTP header extensions using the information provided by the given media frame metadata and media encoded frame metadata.
For video, it may be particularly useful to expose information like whether the frame is a key frame or the spatial layer of the frame if spatial scalability is used.
This information can for instance be sent using a Dependency Descriptor header extension.

Other information like the marker bit or RTP timestamp are also set based on the media frame metadata and media encoded frame metadata.

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
                      |  media frame metadata
                      |  encoded media frame metadata
                      |  encrypted media frame content
                      V
           +----------------------+
           |    Media Decryptor   |
           +----------------------+
                      |  media frame metadata
                      |  encoded media frame metadata
                      |  encoded media frame content
                      V
           +----------------------+
           |   Media Decoder   |
           +----------------------+
                      |  media frame metadata
                      |  media frame content
                      V
                     ... 

Figure 3: A closer look at media content encryption impact on the receiving side.
```

On the receiving side, the depacketizer will regroup the set of RTP packets representing a single media frame based on each RTP packet header information like sequence number, marker bit and timestamp.
From the RTP packet payloads, the receiving side can reconstruct the media frame encrypted content.
It can also identify which decoder to use from the payload type that can be retrieved from the RTP packets.

The receiving side may also identify additional media frame metadata from the RTP header extensions from these RTP packets.
For video content, this can be based on reading the Dependency Descriptor header extension if present.
This is particularly useful for SFUs that need to have a basic understanding of each frame they receive so as to decide to forward it or not and to which endpoint.
By inspecting header extensions, they can do their usual processing without having to inspect the actual media encrypted content.
SFUs may also reassemble the encrypted content for recording purposes or for repacketization of the content.
This can be done as the SFU has the content, the corresponding payload type and the needed metadata from the RTP header extensions.

This is also useful for end recipient processing units as they may want to do some early processing on the content before decryption,
like discarding packets that are not part of a key frame if the processing unit is waiting for a key frame.

Once the encrypted content is reassembled, the processing unit can decrypt it to produce the media frame encoded content.
It can then send the encoded content to the media decoder.


Transmitted media frame metadata
-----------------------

Both intermediaries and end recipient of the RTP packets may need specific information of the content without getting access to the encrypted content.

The information is transmitted as a RTP header extension as the RTP packet payload should be treated as opaque by the SFU.
The amount of information should be limited to what is strictly necessary to typical intermediary tasks since intermediaries may not always be as trusted as individual peers.

For audio, configuration information such as Opus TOC might be useful.
For video, the following configuration information might include:
- Stream configuration information: resolution, quality, frame rate...
- Codec specific configuration information: codec profile like profile_idc...
- Frame specific information: whether the stream is decodable when starting from this frame, whether the frame is skippable...


Proof of concept
=======================

RTP packet generation and reception
-----------------------

One requirement of the proposed approach is that the payload type associated to the media encoder is explicitly conveyed in RTP packets.
One implementation is to prepend on the sender side the encrypted content chunk with the payload type, while the actual RTP payload type would identify that the content is encrypted content.
This makes it easy on the receiver side to identify that the content is encrypted content encoded in a particular format.
The origin encrypted media frame content is computed from the RTP packets by concatenating the RTP payload of the RTP packets after removal of the prepended payload type.

Another requirement of this approach is that SFUs can easily make use of SVC to select the right packets to forward to each end recipient.
This information has to be sent unencrypted.
A potential solution is to send the spatial layer frames using the same SSRC in ascending order, with the same RTP timestamp, and only the last RTP packet of the last spatial layer frame will have the marker bit set to 1.
In addition, to identify how the frame relates to other frames (key frame, spatial layer...), the Dependency Descriptor header extension can be used.
In that case, the first RTP packet of the frame will have its start_of_frame equal to 1 and the last packet will have its end_of_frame equal to 1.
It can be noted that the Dependency Descriptor header extension information can be fueled by the encoded frame metadata, such as the one exposed in WebCodecs SvcOutputMetadata
(see https://docs.google.com/presentation/d/1lFAUSvApbBYfBNJH_xcRW0YjD0aF5T1ZqjyyDJMesJw/edit#slide=id.g1a4ac56601a_3_7 for instance).

SDP negotiation
-----------------------

This section provides a potential illustration of how SDP negotiation can happen, based on the assumption that the codec payload type is sent unencrypted in RTP packets.
A payload type format and the payload type codec and the header extensions useful for packet receiving and routing are negotiated in the SDP O/A for each media type in order to be able to use the RTP encrypted media packetization.
Only the payload types negotiated are allowed to be used as associated payload types.

RTX and FEC procedures apply normally.

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
