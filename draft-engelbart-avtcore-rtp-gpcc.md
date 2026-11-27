---
title: "RTP Payload Format for Geometry-based Point Cloud Compression"
abbrev: "RTP Payload Format for G-PCC"
docname: draft-engelbart-avtcore-rtp-gpcc-latest
category: std
date: {DATE}

ipr: trust200902
area: "Web and Internet Transport"
workgroup: "Audio/Video Transport Core Maintenance"
keyword:
 - RTP
 - G-PCC
 - Payload Format

stand_alone: yes
pi: [toc, sortrefs, symrefs]
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"

venue:
  group: avtcore
  type: Working Group
  mail: avt@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/avt/
  github: mengelbart/draft-engelbart-avtcore-rtp-gpcc
  latest: https://mengelbart.github.io/draft-engelbart-avtcore-rtp-gpcc/draft-engelbart-avtcore-rtp-gpcc.html

author:
 -
    ins: M. Engelbart
    fullname: Mathis Engelbart
    organization: Technical University of Munich
    email: mathis.engelbart@gmail.com
 -
    ins: J. Ott
    fullname: Jörg Ott
    organization: Technical University of Munich
    email: ott@in.tum.de
 -
    ins: L. Kondrad
    fullname: Lukasz Kondrad
    organization: Nokia Technologies
    email: lukasz.kondrad@nokia.com


normative:
   ISO.IEC.23090-9:
      title: "Information technology — Coded representation of immersive media — Part 9: Geometry-based point cloud compression"
      author:
         org: "ISO/IEC"
      date: 2023
      seriesinfo:
         ISO/IEC: 23090-9
      target: https://www.iso.org/standard/78990.html

--- abstract

This memo describes an RTP payload format for geometry-based point cloud
compression (G-PCC). The RTP payload format defined in this document supports
the packetization of one or more data units in an RTP packet payload and the
fragmentation of a single data unit into multiple RTP packets. This memo also
discusses congestion control and rate adaptation for the real-time streaming of
point clouds. Finally, the document defines media type parameters that signal
the profile and level of the G-PCC bitstream, and their use with the Session
Description Protocol (SDP).

--- middle

# Introduction

This document describes the Real-time Transport Protocol (RTP) payload format
for Geometry-based point cloud compression as described in {{ISO.IEC.23090-9}}.
Point clouds are commonly used to represent three-dimensional scans of
environments or objects. The payload format enables the streaming of compressed
point clouds in real-time applications. It applies to various use cases, such as
autonomous vehicles and virtual reality.

In addition to describing the payload format, this document provides guidance
on how to negotiate the parameters to be used with the format, and discusses
congestion control and rate adaptation of point cloud streaming applications.

## Background on Point Clouds

A point cloud is a data structure used to represent three-dimensional scenes.
Point clouds consist of a list of points in three-dimensional space where each
point may optionally be associated with zero or more attributes, such as a color
or reflectance. Point clouds have diverse use cases. For example, a point cloud
can store a 3D representation of the vehicle's surrounding environment in
autonomous cars. Another example is object scanning for the archival of
historical objects or the creation of digital twins of real-world objects for
further analysis. Point clouds are either acquired passively using multiple
camera setups or actively, e.g., using a Lidar sensor to measure distances using
beams of light.

## Overview of the G-PCC Codec

Point clouds can contain large amounts of data, which requires efficient
compression to reduce storage and transmission costs. The Moving Picture Experts
Group (MPEG) has published a Geometry-based Point Cloud Compression (G-PCC)
standard in {{ISO.IEC.23090-9}}. The G-PCC codec takes as input an unordered
list of points, optional attributes, and metadata. All points have the same
number, type, and order of attributes. Attributes can, for example, be the
color, opacity, reflectance, normal vector, material identifier, frame number,
or frame index of the associated point. Compression is defined separately for
geometry and attributes. The attribute coding depends on the decoded geometry.
Thus, geometry encoding and decoding happen before attribute encoding and
decoding.

G-PCC users can choose between two methods for geometry coding and four for
attribute coding. The geometry coding method called *Octree Coding* is a general
compression method, while *Predictive tree* coding specifically targets low
latency applications. The methods for attribute coding are Region Adaptive
Hierarchical Transform (RAHT) coding, Level of Detail (LoD) with Predicting
Transform, LoD with Lifting Transform, and raw attribute data
({{ISO.IEC.23090-9}}, Table 11).

The output of the encoding process is a sequence of Data Units (DUs). Each DU
has a type that describes its layout. {{table-gpcc-unit-type-descriptions}}
lists the ten different types of DUs defined in {{ISO.IEC.23090-9}}.

| DU type | Description                                   |
| 0       | Sequence parameter set data unit              |
| 1       | Geometry parameter set data unit              |
| 2       | Geometry data unit                            |
| 3       | Attribute parameter set data unit             |
| 4       | Attribute data unit                           |
| 5       | Tile inventory data unit                      |
| 6       | Frame boundary marker data unit               |
| 7       | Defaulted attribute data unit                 |
| 8       | Frame-specific attribute properties data unit |
| 9       | User data data unit                           |
{: #table-gpcc-unit-type-descriptions title="G-PCC data unit types"}

Sequence Parameter Set (SPS), Geometry Parameter Set (GPS), and Attribute
Parameter Set (APS) hold the coding parameters of a point cloud sequence, the
geometry coding in use, and the attribute coding in use, respectively.

Geometry and attribute data units contain the coded representation of points
geometry information and points attributes information. Geometry and attribute
data units hold references to their associated parameter sets, and each
referenced parameter set must be available before decoding of the data unit is
possible.

A coded point cloud frame consists of zero or more slices. A slice consists of a
GDU, optional redundant GDUs, and an ADU or defaulted attribute DU per
attribute. A dependent slice requires the preceding slice to be decoded first
({{ISO.IEC.23090-9}}, Section 6.4.5). A tile inventory DU groups slices into
tiles with bounding boxes for spatial random access (Section 6.4.7).

A frame boundary marker DU marks the end of a frame. A defaulted attribute DU
sets one attribute value for all points in a slice. A frame-specific attribute
properties DU overrides SPS attribute properties for a single frame. A user data
DU carries application data that the decoder does not interpret.

Coded point clouds do not have dependencies between frames, i.e., decoding a
point cloud frame is always possible without depending on a previous or
following frame in a sequence. However, future versions of G-PCC might support
inter-frame prediction.

Annex A of {{ISO.IEC.23090-9}} describes profiles and levels of the G-PCC codec.
Decoders can support different profiles and levels. Profiles are subsets of
algorithmic features of the codec specification. A decoder that conforms to a
profile at a specific level must be able to decode all bitstreams that conform
to that profile at a level lower than or equal to the decoder's level
({{ISO.IEC.23090-9}}, Section A.3.2).

# Conventions

{::boilerplate bcp14-tagged}

# Definitions and Abbreviations

This document uses the definitions of {{ISO.IEC.23090-9}}. The following terms
and abbreviations, defined in {{ISO.IEC.23090-9}}, are repeated here for
convenience.

## General G-PCC related terms

{: vspace="0"}
**Attribute**
   : Scalar or vector property associated with each *point* in a *point cloud*.

**Bitstream**
   : Sequence of bits, in the form of encapsulated *data units*, that represents
   a coded *point cloud sequence*.

**Bounding Box**
   : Axis-aligned cuboid defining a spatial region that bounds a set of
   *points*.

**Coded Point Cloud Frame**
   : Coded representation of a *point cloud frame*.

**Data unit**
   : Sequence of bytes conveying a single *syntax structure* of known length.

**Geometry**
   : *Point positions* associated with a set of *points*.

**Point**
   : Fundamental element of a *point cloud* comprising a position specified as
   *Cartesian coordinates* and zero or more *attributes*.

**Point Cloud**
   : Unordered list of *points*.

**Point Cloud Frame**
   : *Point cloud* in a *point cloud sequence*.

**Point Cloud Sequence**
   : Sequence of one or more *point clouds*.

**Point Position**
   : Three dimensional coordinates of a *point*.

**Slice**
   : *Geometry* and *attributes* for part of, or an entire, *coded point cloud
   frame*.

**Syntax element**
   : Element of data represented in the *bitstream*.

**Syntax structure**
   : Zero or more *syntax elements* present together in the *bitstream* in a
   specified order.

**Tile**
   : Set of *slices* identified by a common slice_tag *syntax element* value
   whose *geometry* should be contained within a *bounding box* specified in a
   tile inventory *data unit*.

## Abbreviations

{: vspace="0"}
**ADU**
   : Attribute Data Unit

**APS**
   : Attribute Parameter Set

**DU**
   : Data Unit

**GDU**
   : Geometry Data Unit

**GPS**
   : Geometry Parameter Set

**SPS**
   : Sequence Parameter Set

**TLV**
   : Type-Length-Value

# Payload Format

This section describes the details of the RTP payload format.
{{transmission-mode}} describes the supported transmission mode, {{rtp-header}}
describes the usage of the standard RTP header fields, {{payload-header}}
describes the details of the payload header following the RTP header, and
{{payload-structures}} gives details about the available payload structures and
the specifics of their respective formats.

## Transmission Mode {#transmission-mode}

This payload format supports only single RTP stream transmission. A G-PCC
bitstream MUST be carried in a single RTP stream, as defined in {{?RFC7656}}, on
a single RTP session.

## RTP Header Usage {#rtp-header}

The format of the RTP header is specified in {{!RFC3550}} and replicated in
{{fig-header}} for convenience. This payload format uses the fields of the
header in a manner consistent with that specification.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       sequence number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           synchronization source (SSRC) identifier            |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|            contributing source (CSRC) identifiers             |
|                             ....                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-header title="RTP Header format as specified in RFC 3550"}

{: vspace="0"}

**Marker bit (M): 1 bit**
   : Set to 1 for the last packet of a point cloud frame.

**Payload Type (PT): 7 bits**
   : The assignment of an RTP payload type for this new packet format is outside
   the scope of this document and will not be specified here. The assignment of
   a payload type has to be performed either through the profile used or in a
   dynamic way.

**Sequence Number (SN): 16 bits**
   : Set and used in accordance with {{!RFC3550}}.

**Timestamp: 32 bits**
   : A 90 kHz clock rate MUST be used. The RTP timestamp MUST be set to the
   sampling time of the point cloud frame to which the DUs carried in the packet
   belong. The sampling time of a point cloud frame is the earliest sampling
   time of all points in the frame. For an empty frame, the sampling time is the
   time at which the frame was captured, e.g., the start of the sensor scan that
   produced no points. DUs sent between the last DU of one frame and the first
   DU of the next frame, such as parameter sets, tile inventories, or user data,
   use the RTP timestamp of the next frame.

**Synchronization source (SSRC): 32 bits**
   : Used to identify the source of the RTP packets.

## Payload Header Usage {#payload-header}

The first byte of the payload of an RTP packet is referred to as the payload
header. The payload header consists of the Packet Type (PType in the figures)
and Unit Type fields.

~~~
 0
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|PType|Unit Type|
+-+-+-+-+-+-+-+-+
~~~
{: #fig-payload-header title="General Payload Header"}

{: vspace="0"}
**Packet Type: 3 bits**
  : Type of the packet. This field indicates if the packet is a single unit
  packet, an aggregation packet, or a fragmentation packet. See
  {{payload-structures}} for details. Senders MUST NOT use the values marked as
  reserved in {{table-type-field}}. Receivers MUST discard packets with a
  reserved Packet Type value.

**Unit Type: 5 bits**
  : The type of the following data unit. This field carries the `tlv_type` value
  of the type-length-value encapsulation defined in Annex B of
  {{ISO.IEC.23090-9}} and shown in {{table-gpcc-unit-type-descriptions}}. The
  `tlv_type` field in {{ISO.IEC.23090-9}} is 8 bits long, but only values 0 to
  31 can be carried in the Unit Type field. DUs with a type larger than 31
  cannot be carried by this payload format. Senders MUST NOT send DUs with a
  Unit Type value not listed in {{table-gpcc-unit-type-descriptions}}. Receivers
  MUST discard such DUs.

## Payload Structures {#payload-structures}

Three different RTP packet payload structures are specified: single unit
packets, aggregation packets, and fragmentation packets. A receiver can identify
the payload structure by the Packet Type field of the payload header. The Packet
Type field indicates whether the packet is a single unit packet, an aggregation
packet, or the first, a middle, or the last fragmentation packet of a fragmented
DU. The Packet Type field MUST be set according to {{table-type-field}}.

| Packet Type | Description                                          |
| 000         | Single unit packet {{single-unit-packet}}            |
| 001         | Aggregation packet {{aggregation-packet}}            |
| 010         | First fragmentation packet {{fragmentation-packet}}  |
| 011         | Middle fragmentation packet {{fragmentation-packet}} |
| 100         | Last fragmentation packet {{fragmentation-packet}}   |
| 101         | Reserved                                             |
| 110         | Reserved                                             |
| 111         | Reserved                                             |
{: #table-type-field title="Packet Type field values"}

The following sections explain the details and variations from this format for
the three packet types.

### Single Unit Packet {#single-unit-packet}

A single unit packet contains only one data unit. The first byte in the packet
is the payload header, followed by the payload data. The data unit extends until
the end of the packet, excluding any RTP padding. The DU is carried without its
TLV header. The Packet Type field of a single unit packet MUST be set to zero
(binary `000`).

{{fig-single-unit-packet}} shows an example of a single unit packet.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| PayloadHeader |                                               |
+-+-+-+-+-+-+-+-+                                               |
|                           Data Unit                           |
|                                                               |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               :...OPTIONAL RTP padding        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-single-unit-packet title="Single unit packet payload"}

### Aggregation Packet {#aggregation-packet}

An aggregation packet contains two or more DUs. Aggregation packets can reduce
packetization and header overhead for small DUs such as parameter sets. Each DU
is prefixed with a separate payload header and a Size field, forming an
aggregation unit. The Size field is a 16-bit unsigned integer in network byte
order indicating the length of the following DU in bytes, excluding the payload
header and the Size field. DUs are carried without their TLV header. There is no
padding between the DUs of an aggregation packet, i.e., the payload header of a
DU directly follows the last byte of the preceding DU.

An aggregation packet MUST carry at least two DUs. Aggregation packets MAY carry
more than two DUs. The total amount of data in an aggregation packet MUST fit
into an IP packet, and the size SHOULD be chosen so that the resulting IP packet
is smaller than the MTU size in order to avoid IP layer fragmentation. An
aggregation packet MUST NOT contain fragments of DUs.

An aggregation packet MUST NOT carry DUs of different point cloud frames.

The Packet Type field of every aggregation unit in an aggregation packet MUST be
set to the value one (binary `001`).

A receiver parses the DUs of an aggregation packet in order, after removing any
RTP padding. If a payload header with a Packet Type field other than `001` is
found, if the Size field of a DU indicates more bytes than remain in the packet,
or if bytes remain after the last DU that do not form a complete payload header,
Size field, and DU, the receiver MUST discard the remainder of the packet,
starting with the affected DU. DUs preceding it in the packet are not affected.

{{fig-aggregation-header}} shows the format of an aggregation unit.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|PType|Unit Type|            Size               |   Data Unit  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-aggregation-header title="Aggregation unit format"}

An example of an aggregation packet with two aggregation units is shown in
{{fig-aggregation-format}}.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| PayloadHeader |            Size               |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               +
|                                                               |
|                           Data Unit                           |
|                                               +-+-+-+-+-+-+-+-+
|                                               | PayloadHeader |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Size              |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                                                               |
|                           Data Unit                           |
|                                                               |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               :...OPTIONAL RTP padding        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-aggregation-format title="Aggregation packet payload with two aggregation units"}

### Fragmentation Packet {#fragmentation-packet}

Fragmentation packets allow a single DU to be split into multiple RTP packets
without cooperation from the G-PCC encoder. A DU that does not fit into a single
RTP packet without exceeding the path MTU SHOULD be fragmented. Fragments of the
same DU MUST be sent in consecutive order with ascending RTP sequence numbers
(with no other RTP packets within the same RTP stream being sent between the
first and last fragment).

Aggregation packets MUST NOT be fragmented. Fragmentation packets MUST NOT be
nested, i.e., a fragmentation packet cannot contain a fragment of another
fragmentation packet.

A fragmentation packet consists of the payload header followed by a fragment of
the DU. The fragment extends until the end of the packet, excluding any RTP
padding. It is a sequence of consecutive bytes of the DU, without the TLV
header. The DU is the concatenation of the fragments of all fragmentation
packets of the series, in RTP sequence number order.
{{fig-fragmentation-packet}} shows the format of a fragmentation packet.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| PayloadHeader |                                               |
+-+-+-+-+-+-+-+-+                                               |
|                         DU Fragment                           |
|                                                               |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               :...OPTIONAL RTP padding        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-fragmentation-packet title="Fragmentation packet payload"}

The Packet Type field of the first fragmentation packet of a fragmented DU MUST
be set to two (binary `010`), the Packet Type field of the last fragmentation
packet MUST be set to four (binary `100`), and the Packet Type field of all
fragmentation packets between the first and the last MUST be set to three
(binary `011`).

The Unit Type field of every fragmentation packet MUST be set to the type of the
fragmented DU.

A fragmented DU MUST be carried in at least two fragmentation packets, i.e., a
DU MUST NOT be sent as a single fragmentation packet. Each fragmentation packet
MUST carry a non-empty fragment.

The RTP timestamp of all fragmentation packets of a fragmented DU MUST be set to
the same value: the RTP timestamp of the point cloud frame to which the DU
belongs, as defined in {{rtp-header}}.

A receiver reassembles a DU from a series of fragmentation packets that starts
with a first fragmentation packet (`010`), continues with zero or more middle
fragmentation packets (`011`), and ends with a last fragmentation packet
(`100`), all with consecutive RTP sequence numbers, the same RTP timestamp, and
the same Unit Type. A receiver MUST treat the series as incomplete and MUST
discard the fragments received so far if any of the following occurs:

* a gap in RTP sequence numbers between the first and the last fragmentation
  packet,
* a middle or last fragmentation packet that is not preceded by a first
  fragmentation packet of the same series,
* a first fragmentation packet, or a packet other than a fragmentation packet,
  before the last fragmentation packet of the series,
* a change in RTP timestamp or Unit Type within the series.

A receiver MUST NOT pass incomplete DUs to the decoder.

# Packetization Rules and Depacketization Rules

## Packetization Rules

* DUs MUST be transmitted in decoding order, i.e., in the order in which they
  appear in the G-PCC bitstream as defined in Annex B of {{ISO.IEC.23090-9}}.
  The RTP sequence number of a packet carrying a DU MUST be higher (modulo
  2^16) than the RTP sequence number of any packet carrying a DU that precedes
  it in decoding order.
* Within an aggregation packet, DUs MUST appear in decoding order.
* A DU of a small size SHOULD be encapsulated in an aggregation
  packet together with one or more other DUs in order to avoid the
  unnecessary packetization overhead for small DUs. For example,
  parameter sets are typically small and can often be aggregated without
  violating MTU size constraints.
* For carrying exactly one DU in an RTP packet, a single unit
  packet MUST be used.

## Depacketization Rules

A receiver MUST pass DUs to the decoder in RTP sequence number order. Within an
aggregation packet, DUs MUST be passed to the decoder in the order in which they
appear in the packet. To do so, a receiver performs the following steps:

* All normal RTP mechanisms related to buffer management apply. In particular,
  duplicated or outdated RTP packets (as indicated by the RTP sequence number
  and the RTP timestamp) are removed, and packets are reordered by RTP sequence
  number.
* The DUs are extracted from each packet according to the Packet Type field
  ({{payload-structures}}). Fragments of a DU are reassembled as described in
  {{fragmentation-packet}}, which also defines the handling of incomplete DUs.
* Each DU is passed to the decoder as a TLV encapsulation structure as defined
  in Annex B of {{ISO.IEC.23090-9}}, with `tlv_type` set to the Unit Type of the
  payload header and `tlv_num_payload_bytes` set to the length of the DU in
  bytes.

To determine the exact time for decoding, factors such as a possible intentional
delay to allow for proper inter-stream synchronization need to be factored in.
How long a receiver waits for reordered or retransmitted packets before
considering them lost is implementation dependent.

# Payload Format Parameters

This section defines the media type to use with the payload format and its
parameters.

## Media Type Definition {#media-type}

This section describes the media type registration following {{!RFC6838}} and
{{!RFC4855}}.

{: vspace="0"}
**Type name**:
  : application

**Subtype name**:
  : gpcc

**Required parameters**:
  : profile-compatibility, level-idc. See {{required-parameters-definition}}.

**Optional parameters**:
  : N/A

**Encoding considerations**:
  : This media type is framed and contains binary data (see {{Section 4.8 of
  RFC6838}}).

**Security considerations**:
  : See {{security-considerations}}.

**Interoperability considerations**:
  : N/A

**Published specification**:
  : RFC XXXX and {{ISO.IEC.23090-9}}.

**Applications that use this media type**:
  : Any application that relies on G-PCC-based point cloud transmission over
  RTP.

**Fragment identifier considerations**:
  : N/A

**Additional information**:
  : N/A

**Person & email address to contact for further information**:
  : Mathis Engelbart (mathis.engelbart@gmail.com)

**Intended usage**:
  : COMMON

**Restrictions on usage**:
  : This media type depends on RTP framing and is only defined for transfer via
  RTP {{!RFC3550}}. Transfer within other framing protocols is not defined.

**Author**:
  : See the Authors' Addresses section of RFC XXXX.

**Change controller**:
  : IETF <avt@ietf.org>

**Provisional registration? (standards tree only)**:
  : No

## Required Parameters Definition {#required-parameters-definition}

{: vspace="0"}
**profile-compatibility**:
  : A base16 {{!RFC4648}} (hexadecimal) representation of three bytes, i.e.,
  exactly six hexadecimal digits. The value is case-insensitive. The first 22
  bits, most significant bit first, are, in this order,
  `simple_profile_compliant`, `dense_profile_compliant`,
  `predictive_profile_compliant`, `main_profile_compliant`, and
  `reserved_profile_18bits` as defined in the sequence parameter set of
  {{ISO.IEC.23090-9}}, Section 7.3.2.1. The bits corresponding to
  `reserved_profile_18bits` MUST be set to 0 and MUST be ignored by receivers.
  The last two bits MUST be set to 0 and MUST be ignored by receivers. They do
  not correspond to the `slice_reordering_constraint` and
  `unique_point_positions_constraint` flags at the same positions in the SPS.
  When used to describe a bitstream, the bits are set to the values of the
  corresponding syntax elements in the SPS of the bitstream. When used to
  describe receiver capabilities, a bit set to 1 indicates that the receiver
  supports the corresponding profile. For example, `800000` indicates the Simple
  profile, and `100000` indicates the Main profile.

**level-idc**:
  : A decimal representation of the `level_idc` syntax element of the SPS as
  defined in {{ISO.IEC.23090-9}}, Section 7.3.2.1, i.e., 20 times the level
  number (e.g., `80` for level 4). The value is in the range 0 to 255. Only
  values corresponding to levels defined in {{ISO.IEC.23090-9}}, Section A.4.1
  (currently `80` and `120`) are valid. When used to describe a bitstream, it is
  the level to which the bitstream conforms. When used to describe receiver
  capabilities, it is the highest level that the receiver supports for all
  profiles indicated in profile-compatibility. Levels are compared by their
  `level_idc` value as specified in {{ISO.IEC.23090-9}}, Section A.4.1. A
  receiver that supports different levels for different profiles can offer
  multiple payload types, each with a single level.

## SDP Parameters

### Mapping of Payload Type Parameters to SDP

The media type application/gpcc string is mapped to fields in the Session
Description Protocol (SDP) {{!RFC8866}} as follows:

* The media name in the "m=" line of SDP MUST be application.
* The encoding name in the "a=rtpmap" line of SDP MUST be gpcc (the media
  subtype).
* The clock rate in the "a=rtpmap" line MUST be 90000.
* The parameters *profile-compatibility* and *level-idc* MUST be included in the
  "a=fmtp" line of SDP. The fmtp line is expressed as a media type string, in
  the form of a semicolon-separated list of parameter=value pairs.

### Examples

The following example shows the media description of an offer to send a stream
conforming to the Simple profile at level 4:

~~~
m=application 49170 RTP/AVP 96
a=rtpmap:96 gpcc/90000
a=fmtp:96 profile-compatibility=800000;level-idc=80
a=sendonly
~~~

In the following example, the offerer is able to receive bitstreams conforming
to the Dense or Main profile up to level 6 on payload type 96, and bitstreams
conforming to the Simple profile up to level 4 on payload type 97:

~~~
m=application 49170 RTP/AVP 96 97
a=rtpmap:96 gpcc/90000
a=fmtp:96 profile-compatibility=500000;level-idc=120
a=rtpmap:97 gpcc/90000
a=fmtp:97 profile-compatibility=800000;level-idc=80
a=sendrecv
~~~

The answerer is only able to receive bitstreams conforming to the Main profile
up to level 4. It accepts payload type 96 and removes payload type 97:

~~~
m=application 49172 RTP/AVP 96
a=rtpmap:96 gpcc/90000
a=fmtp:96 profile-compatibility=100000;level-idc=80
a=sendrecv
~~~

The offerer sends a bitstream conforming to the Main profile at level 4 or
lower. The answerer sends a bitstream conforming to the Dense or Main profile
at level 6 or lower.

### Offer/Answer Considerations

When the payload format is offered over RTP using SDP in an Offer/Answer model
as described in {{!RFC3264}}, the following rules apply to the parameters
profile-compatibility and level-idc:

* In an offer or answer for a stream with direction sendrecv or recvonly, the
  parameters declare the profiles and level that the offerer or answerer is able
  to receive. The values in the answer are independent of the values in the
  offer.
* In an offer for a stream with direction sendonly, the parameters describe the
  bitstream that the offerer will send. An answerer that is able to receive the
  bitstream MUST use the same values in the answer. Otherwise, it MUST remove
  the payload type from the answer.
* In an answer for a stream with direction sendonly, the parameters describe the
  bitstream that the answerer will send. The values MUST fulfill the conditions
  below for the profiles and level declared in the offer. An answerer that is
  not able to send such a bitstream MUST remove the payload type from the
  answer.
* A bitstream sent to a receiver MUST indicate conformance to at least one of
  the profiles indicated by the receiver in profile-compatibility, and MUST
  conform to a level that is lower than or equal to the level indicated by the
  receiver in level-idc.
* An offerer or answerer that is not able to send a bitstream fulfilling these
  conditions MUST NOT send on the payload type.
* For multicast streams, the answerer MUST use the values of the offer or remove
  the payload type, as all receivers receive the same bitstream.

### Declarative SDP Considerations

When G-PCC over RTP is offered with SDP in a declarative style, as in Real Time
Streaming Protocol (RTSP) {{?RFC7826}} or Session Announcement Protocol (SAP)
{{?RFC2974}}, the following considerations are necessary.

* The parameters profile-compatibility and level-idc describe the bitstream that
  is sent, not the capabilities for receiving streams.
* A receiver MUST reject (RTSP) or not participate in (SAP) the session if it
  does not support at least one of the indicated profiles at the indicated level
  or higher, or if it does not support any other parameter or value provided. It
  falls on the creator of the session to use values that are expected to be
  supported by the receiving application.

# Congestion Control

Congestion control for RTP SHALL be used in accordance with {{!RFC3550}}, and
with any applicable RTP profile: e.g., {{?RFC3551}}.  If best-effort service is
being used, users of this payload format MUST monitor packet loss to ensure that
the packet loss rate is within acceptable parameters. Circuit Breakers
{{!RFC8083}} is an update to RTP {{!RFC3550}} that defines criteria for when one
is required to stop sending RTP Packet Streams. RTP circuit breakers
{{!RFC8083}} MUST be implemented and followed.

Applications SHOULD also use a congestion control algorithm for interactive
real-time media, such as {{?RFC8298}} or {{?RFC8698}}, which can use the RTCP
feedback defined in {{?RFC8888}}.

When real-time encoding is used, the sender can adapt the bitrate by changing
the frame rate, the geometry precision (position quantization or octree depth),
the attribute quantization parameters, the set of attributes sent, or the point
density of the input.

In addition, if the payload is not encrypted or the middlebox is part of the
SRTP security context, a Selective Forwarding Middlebox ({{?RFC7667}}, Section
3.7) can reduce the bitrate by dropping selected DUs. Some of the rules below
require parsing the SPS and GDU header in addition to the payload header. The
middlebox needs to rewrite sequence numbers, RTCP reports, aggregation packets,
and the marker bit, so that dropped DUs do not appear as packet loss.

Frames are coded independently ({{ISO.IEC.23090-9}}, Section 6.4.4), so a
middlebox can drop all DUs of a frame except parameter sets and tile
inventories. When dropping only some DUs of a frame, the following dependencies
apply ({{ISO.IEC.23090-9}}):

* Parameter sets are needed to decode all DUs that reference them. Middleboxes
  SHOULD NOT drop them.
* A tile inventory applies to all following frames until it is replaced by
  another tile inventory (Section 7.4.2.4). Middleboxes SHOULD NOT drop it.
* Frame-specific attribute properties DUs override attribute properties of the
  SPS, such as scale and offset, for a single frame. Without them, decoded
  attribute values of the frame can be misinterpreted. Middleboxes SHOULD NOT
  drop them.
* ADUs of a slice cannot be decoded without a GDU of the slice. If all GDUs of a
  slice are dropped, its ADUs SHOULD be dropped as well.
* Without its ADUs, the geometry of a slice remains decodable with default
  attribute values (Section 8.3.4), but the bitstream no longer conforms
  (Section 6.4.5) unless the ADUs are replaced by defaulted attribute DUs.
* When `slice_reordering_constraint` in the SPS is 1, removing slices can make
  following slices undecodable (Sections 7.4.2.1.1 and 7.4.3.2). Middleboxes
  SHOULD NOT drop slices of such bitstreams. In addition, when
  `entropy_continuation_enabled` in the SPS is 1, parsing of a GDU or ADU can
  depend on DUs of preceding slices (Section 11.6). Middleboxes SHOULD NOT drop
  GDUs or ADUs of such bitstreams. `entropy_continuation_enabled` can only be 1
  when `slice_reordering_constraint` is 1 (Section 7.4.2.1.1).
* Middleboxes SHOULD drop either all or none of the fragments of a DU.

# IANA Considerations

IANA is requested to register the media type application/gpcc, as specified in
{{media-type}}, in the "Media Types" registry
<https://www.iana.org/assignments/media-types>.

# Security Considerations {#security-considerations}

RTP packets using the payload format defined in this specification are subject
to the security considerations discussed in the RTP specification {{!RFC3550}},
and in any applicable RTP profile such as RTP/AVP {{?RFC3551}}, RTP/AVPF
{{?RFC4585}}, RTP/SAVP {{?RFC3711}}, or RTP/SAVPF {{?RFC5124}}.  However, as
"Securing the RTP Protocol Framework: Why RTP Does Not Mandate a Single Media
Security Solution" {{?RFC7202}} discusses, it is not an RTP payload format's
responsibility to discuss or mandate what solutions are used to meet the basic
security goals like confidentiality, integrity, and source authenticity for RTP
in general.  This responsibility lies on anyone using RTP in an application.
They can find guidance on available security mechanisms and important
considerations in "Options for Securing RTP Sessions" {{?RFC7201}}.
Applications SHOULD use one or more appropriate strong security mechanisms.  The
rest of this Security Considerations section discusses the security impacting
properties of the payload format itself.

The receiver-side computational complexity of G-PCC decoding is not uniform. A
small amount of coded data can represent a large number of points. For example,
the syntax elements `occ_dup_point_cnt`, `direct_dup_point_cnt`, and
`ptn_dup_point_cnt` can each signal up to MaxSlicePoints - 1 points with
identical positions ({{ISO.IEC.23090-9}}, Table A.9), where MaxSlicePoints is
2^20 for all levels defined in {{ISO.IEC.23090-9}}, Table A.2. The level limits
apply per slice and do not limit the number of slices or points per frame, the
frame rate, or the bitrate. As stated in {{ISO.IEC.23090-9}}, Section 7.4.3.3,
decoders cannot rely on bitstream conformance to prevent overflow of
implementation buffers. Receivers SHOULD enforce local limits, e.g., on the
number of points per slice and per frame, the number of slices per frame, and
the size of DUs, and SHOULD discard data that exceeds them.

The TLV encapsulation allows DUs of up to 2^32 - 1 bytes, and fragmented DUs are
reassembled in receiver memory. Receivers SHOULD limit the size of DUs they
reassemble and SHOULD discard incomplete DUs that are not completed within an
implementation-specific time. Receivers MUST validate the Size field of each DU
in an aggregation packet against the remaining length of the packet as described
in {{aggregation-packet}}.

This RTP payload format does not contain active content. User data DUs carry
data not specified by {{ISO.IEC.23090-9}}, which receivers that interpret it
MUST treat as untrusted input.

Point clouds can capture people and private spaces. Applications SHOULD use
confidentiality mechanisms appropriate for the captured content.

# RFC Editor Considerations

Note to RFC Editor: This section may be removed after carrying out all the
instructions of this section.

Please replace "RFC XXXX" with the RFC number of this document.

--- back

# Acknowledgments
{:numbered="false"}

The authors used a large language model to assist with drafting and editing
text in this document. The authors reviewed all generated text and take full
responsibility for the content of this document.
