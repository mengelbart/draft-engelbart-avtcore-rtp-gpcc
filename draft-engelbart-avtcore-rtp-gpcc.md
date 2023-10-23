---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "RTP Payload Format for Geometry-based Point Cloud Compression"
abbrev: "RTP Payload Format for G-PCC"
category: info

docname: draft-engelbart-avtcore-rtp-gpcc-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: Applications and Real-Time Area
workgroup: AVTCORE
keyword:
 - RTP
 - G-PCC
 - Payload Format
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
    organization: Technical University Munich
    email: mathis.engelbart@tum.de
 -
    ins: J. Ott
    fullname: Jörg Ott
    organization: Technical University Munich
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

informative:

--- abstract

This memo describes an RTP payload format for geometry-based point cloud
compression (G-PCC) ({{ISO.IEC.23090-9}}). The RTP payload format defined in
this document supports the packetization of one or more data units in an RTP
packet payload and the fragmentation of a single data unit into multiple RTP
packets. This memo also describes congestion control for a practical solution
for the real-time streaming of point clouds.
Finally, the document defines parameters that may be used to select
optional features of the payload format or signall properties of the RTP stream.
The parameters can be used with Session Description Protocol (SDP).

--- middle

# Introduction

This document describes the Real-time Transport Protocol (RTP) payload format
for Geometry-based point cloud compression as described in {{ISO.IEC.23090-9}}.
Point clouds are commonly used to represent three-dimensional scans of
environments or objects. The payload format enables the streaming of compressed
point clouds in real-time applications. It applies to various use cases, such as
autonomous vehicles and virtual reality.

In addition to describing the payload format, this document also includes
examples of the payload format, provides guidance on how to negotiate the
parameters to be used with the format, and discusses congestion control and rate
adaptation of point cloud streaming applications.

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
color, opacity, reflectance, or frame number of the associated point.
Compression is defined separately for geometry and attributes. The attribute
coding depends on the decoded geometry. Thus, geometry encoding and decoding
happen before attribute encoding and decoding.

G-PCC users can choose between two methods for geometry coding and three for
attribute coding. The geometry coding method called *Octree Coding* is a general
compression method, while *Predictive tree* coding specifically targets low
latency applications. The methods for attribute coding are called Region
Adaptive Hierarchical Transform (RAHT) coding, Predicting Transform, and Lifting
Transform.

The output of the encoding process is a sequence of Data Units (DUs). Each
DU has a type that describes its layout.
{{table-gpcc-unit-type-descriptions}} lists the ten different types of DUs
defined in {{ISO.IEC.23090-9}}.

| Type   | Description                                   |
| 0      | Sequence parameter set data unit              |
| 1      | Geometry parameter set data unit              |
| 2      | Geometry data unit                            |
| 3      | Attribute parameter set data unit             |
| 4      | Attribute data unit                           |
| 5      | Tile inventory data unit                      |
| 6      | Frame boundary marker data unit               |
| 7      | Defaulted attribute data unit                 |
| 8      | Frame-specific attribute properties data unit |
| 9      | User data data unit                           |
{: #table-gpcc-unit-type-descriptions title="G-PCC data unit types"}

Sequence Parameter Set (SPS), Geometry Parameter Set (GPS), and Attribute
Parameter Set (APS) hold the coding parameters of a point cloud sequence, the
geometry coding in use, and the attribute coding in use, respectively.

Geometry and attribute data units contain the coded representation of points geometry information and
points attributes information. Geometry and attribute data units hold references to their
associated parameter sets, and each referenced parameter set must be available
before decoding of the data unit is possible.

Coded point clouds do not have dependencies between frames, i.e., decoding a
point cloud frame is always without depending on a previous or following frame
in a sequence. However, future versions of G-PCC might support inter-frame
prediction.

Annex A of {{ISO.IEC.23090-9}} describes profiles and levels of the G-PCC codec.
Decoders can support different profiles and levels. Profiles are subsets of
algorithmic features of the codec specification. A decoder that supports a
specific profile must be able to decode a bitstream conforming to that profile.

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
   : A sequence of bits.

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
   : Sequence of one or more *pont clouds*.

**Poisition**
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

This section describes the details of the RTP payload format. {{rtp-header}}
describes the usage of the standard RTP header fields, {{payload-header}}
describes the details of the payload header following the RTP header, and
{{payload-structures}} gives details about available packetization modes and the
specifics of their respective formats.

## Transmission Modes

> **TODO**: Do we need this section on transmission modes similar to other payload formats to define SRST, MRST, and MRMT?

## RTP Header Usage {#rtp-header}

The format of the RTP header is specified in {{!RFC3550}} and replicated in {{fig-header}} for convenience. This payload format uses the fields of the header in a manner consistent with that specification.

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

> **TODO**: if we add a dedicated subsection on SRST/SRMT/MRMT, add the
> following statement:
>
> When MRST or MRMT is in use, if an access unit appears in multiple RTP
> streams, the marker bit is set on each RTP stream's last packet of the access
> unit.

{: vspace="0"}

**Marker bit (M): 1 bit**
   : Set to 1 for the last packet of a point cloud frame carried in the current RTP stream.

**Payload Type (PT): 7 bits**
   : The assignment of an RTP payload type for this new packet format is outside
   the scope of this document and will not be specified here. The assignment of
   a payload type has to be performed either through the profile used or in a
   dynamic way.

**Sequence Number (SN): 16 bits**
   : Set and used in accordance with {{!RFC3550}}.

**Timestamp: 32 bits**
   : The RTP timestamp is set to the sampling timestamp of the content. A 90 kHz
   clock rate MUST be used. The sampling timestamp of the content is the
   reconstruction time. It denotes the earliest sampling time of all points in the
   point cloud frame to which the data unit transmitted in the packet belongs.
   If the data unit has no timing properties (e.g., parameter sets), the RTP
   timestamp is set to the RTP timestamp of the first data unit that references
   the parameter set. If the packet is an aggregation unit packet, all data
   units MUST have the same timestamp.

**Synchronization source (SSRC): 32 bits**
   : Used to identify the source of the RTP packets.

## Payload Header Usage {#payload-header}

The first bytes of the payload of an RTP packet are referred to as the payload
header. The payload header consists of a packet type and unit type field.

~~~
 0
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
| Typ |Unit-Type|
+-+-+-+-+-+-+-+-+
~~~
{: #fig-payload-header title="General Payload Header"}

{: vspace="0"}
**Typ: 3 bits**
  : Type of the packet. This field indicates if the packet is a single unit
  packet, an aggregation unit packet, or a fragmentation unit packet. See
  {{payload-structures}} for details.

**Unit-Type: 5 bits**
  : The type of the following data unit. This field specifies the type field
  from the type-length-value encapsulation defined in Annex-B of G-PCC
  {{ISO.IEC.23090-9}} and shown in {{table-gpcc-unit-type-descriptions}}.

## Payload Structures {#payload-structures}

Three different RTP packet payload structures are specified: Single Unit
Packets, Aggregation Unit Packets, and Fragmentation Unit Packets. A receiver
can identify the payload structure by the type field of
the payload header. The type field indicates whether the unit is a single unit,
an aggregation unit, or the first, last, or middle part of a fragmentation unit
packet. The type field MUST be set according to {{table-type-field}}.

| Type   | Description                                                         |
| 000    | Single Unit Packet {{single-unit}}                                  |
| 001    | Aggregation Unit Packet {{aggregation-packet}}                        |
| 010    | First packet of an fragmentation unit packet {{fragmentation-unit}} |
| 011    | Fragmentation unit packet {{fragmentation-unit}}                    |
| 100    | Last packet of an fragmentation unit packet {{fragmentation-unit}}  |
| 101    | Reserved                                                            |
| 110    | Reserved                                                            |
| 111    | Reserved                                                            |
{: #table-type-field title="Type field values"}

The following sections explain the details and variations from this format for
the three packet types.

### Single Unit {#single-unit}

A Single Unit packet contains only one data unit. The first byte in the packet
is the payload header, followed by the payload data. The data unit extends until
the end of the packet. The packet type field of a single unit packet MUST be set
to zero (binary `000`).

{{fig-single-unit-payload}} shows an example of a single unit packet.

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
{: #fig-single-unit-payload title="Single unit packet payload"}

### Aggregation Packet {#aggregation-packet}

An aggregation packet contains two or more DUs. Aggregation
packets can reduce packetization and header overhead for small DUs
such as parameter sets. Each DU is prefixed with a separate
payload header and an additional length field. The length field is of variable
size and indicates the length of the DU. The two most significant
bits of the length field encode the base-2 logarithm of the size of the length
field in bytes as defined in Section 16 of {{!RFC9000}}. Thus, the length field
can have 1, 2, 4, or 8 bytes. The receiver can split the packet payload into
individual units by reading the length byte.

An aggregation packet MUST carry at least two DUs. Aggregation
packets MAY carry more than two DUs. The total amount of data in
an aggregation packet MUST fit into an IP packet, and the size SHOULD be chosen
so that the resulting IP packet is smaller than the MTU size in order to avoid
IP layer fragmentation. An aggregation packet MUST NOT contain any fragmentation
units.

An aggregation packet MUST NOT carry DUs of different point cloud
frames, i.e., all DUs included in an aggregation packet MUST have
the same timestamp.

The packet type field of every DU in an aggregation packet MUST be
set to the value one (binary `001`).

{{fig-aggregation-header}} shows the extended payload header format of syntax
structures in aggregation packets.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Typ=1|Unit-Type|                VarInt Length                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-aggregation-header title="Aggregation Unit Payload"}

An example of an aggregation unit packet with two data unit payloads is shown in
{{fig-aggregation-format}}.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| PayloadHeader |                VarInt Length                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                           Data Unit                           |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| PayloadHeader |                VarInt Length                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                           Data Unit                           |
|                                                               |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               :...OPTIONAL RTP padding        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-aggregation-format title="Packet Payload"}

### Fragmentation Unit {#fragmentation-unit}

Fragmentation units allow fragmentation of single DUs into
multiple RTP packets without cooperation from the G-PCC encoder. Fragments of
the same DU MUST be sent in consecutive order with ascending RTP
sequence numbers (with no other RTP packets within the same RTP stream being
sent between the first and last fragment).

Aggregation packets MUST NOT be fragmented. Fragmentation units MUST NOT be
nested, i.e., a fragmentation unit cannot contain a subset of another
fragmentation unit.

The timestamp of all fragmentation units MUST be set to the same value: the
timestamp of the DU that is carried in the fragmentation unit
packets.

The type field of the first packet of a series of fragmentation units MUST be
set to 2 (binary `010`), the type field of the last packet in the series MUST be
set to four (binary `100`), and the type field of all packets between the first
and last packet of the fragmented unit MUST have the value three (binary `011`).

# Packetization Rules and Depacketization Rules

* A DU of a small size SHOULD be encapsulated in an aggregation
  packet together with one or more other DUs in order to avoid the
  unnecessary packetization overhead for small DUs. For example,
  parameter sets are typically small and can often be aggregated with without
  violating MTU size constraints.
* For carrying exactly one DU in an RTP packet, a single unit
  packet MUST be used.

The de-packetization process is implementation dependent. Therefore, the
following de-packetization rules SHOULD be taken as an example.

* All normal RTP mechanisms related to buffer management apply. In particular,
  duplicated or outdated RTP packets (as indicated by the RTP sequence number
  and the RTP timestamp) are removed. To determine the exact time for decoding,
  factors such as a possible intentional delay to allow for proper inter-stream
  synchronization must be factored in.

# Payload Format Parameters

This section defines the media type to use with the payload format and the
optional parameters that can be used with it.

## Media Type Definition {#media-type}

> **Note**: Template from {{Section 10 of !RFC4288}}

{: vspace="0"}
**Type name**:
  : application

**Subtype name**:
  : GPCC

**Required Parameters**:
  : None

**Optional Parameters**:
  : profile-level-id **TODO**: add list from section {{optional-parameters-definition}}

**Encoding considerations**:
  : This type is only defined for transfer via RTP {{!RFC3550}}.

**Security considerations**:
  : See {{security-considerations}}.

**Interoperability considerations**:
  : N/A

**Published specification**:
  : Please refer to <**TODO**: Reference this document> and
  {{ISO.IEC.23090-9}}.

**Applications that use this media type**:
  : Any application that relies on GPCC-based point cloud transmission over RTP

**Fragment identifier considerations**:
  : N/A

**Additional information**:
  : N/A

**Person & email address to contact for further information**:
  : Mathis Engelbart (mathis.engelbart@gmail.com)

**Intended usage**:
  : COMMON

**Restrictions on usage**:
  : N/A

**Author**:
  : See Authors' Addresses section of <**TODO**: Reference this document>

**Change controller**:
  : IETF <avtcore@ietf.org>

## Optional Parameters Definition {#optional-parameters-definition}

**profile-level-id**:
  : The profile-level-id is a base16 {{!RFC4648}} (hexadecimal) representation
  of one byte containing the flags indicating support or conformance to the
  G-PCC profiles and levels. The first four bits are flags indicating support
  for the profiles *Simple*, *Predictive*, *Dense* and *Main*, while the last
  for bits indicate the highest level that the decoder supports or a bitstream
  conforms to.

**pcc-resolution**:
: Describes the resolution of the point cloud as number of lines, number of
points per line and depth resolution.

> \<#lines\> \<#points-per-line\> \<#depth-resolution\>

**pcc-coverage**:
: The field of view coverage of the sensor described by the horizontal and
vertical angles.

> \<h-angle\> \<v-angle\>

**pcc-anchor**:
: An anchor reference point for the point cloud, relative to which coordinates
are given. This can be useful to relate multiple point clouds to each other.

> \<x\> \<y\> \<z\>

**pcc-orientation**:
: The orientation of the point cloud sensor.

> \<tilt\> \<pan\>

**pcc-position**:
: The location of the LiDAR or point cloud sensor relative to *pcc-anchor* and
  *pcc-orientation*:

> \<delta-x\> \<delta-y\> \<delta-z\> \<delta-tilt\> \<delta-pan\>

**pcc-point-attr**:
: A list of attributes associated with each point. Example: color,
  reflectivity.

## SDP Parameters

### Mapping of Payload Type Parameters to SDP

The media type application/GPCC string is mapped to fields in the Session
Description Protocol (SDP) {{!RFC8866}} as follows:

> **TODO**: Add all parameters from above

* The media name in the "m=" line of SDP MUST be application.
* The encoding name in the "a=rtpmap" line of SDP MUST be GPCC (the media
  subtype).
* The clock rate in the "a=rtpmap" line MUST be 90000.
* The optional parameters *profile-level-id*, ... when present, MUST be included
  in the "a=fmtp" line of SDP. The fmtp line is expressed as a media type
  string, in the form of a semicolon-separated list of parameter=value pairs.

### Example

~~~
m=application 54321 RTP/AVPF 95
a=rtpmap:95 GPCC/90000
a=fmtp:95 profile-level-id=84; // 0x84 => 0b10000100 => simple profile, level 4
          --
          -- point cloud resolution
          pcc-resolution: #lines #points-per-line #depth-resolution;
          --
          -- field of view
          pcc-coverage: h-angle v-angle;
          --
          -- optional anchor point 
          pcc-anchor: <type> <type-specific format> "mm" x y z | "gps" lat lon alt;
          pcc-orientation: tilt pan;
          --
          -- where is this lidar located (relative to possible others)
          pcc-position: delta-x, delta-y, delta-z delta-tilt delta-pan;
          --
          -- which attributes are encoded per point: list of properties | profile
          pcc-point-attr: "attr" color, reflectance;
~~~

### Offer/Answer Considerations

> **TODO**: Are theses rules correct for G-PCC and do we need considerations for
> other parameters?

When the payload format is offered over RTP using SDP in an Offer/Answer model
as described in {{!RFC3264}} for negotiation for unicast usage, the following
limitations and rules apply:

* The parameter identifying a media format configuration for G-PCC is
  profile-levle-id. This media format configuration parameter MUST be used
  symmetrically; that is, the answerer MUST either maintain this configuration
  parameter or remove the media format (payload type) completely if it is not
  supported.
* The G-PCC stream sent by either the offerer or the answerer MUST be encoded
  with a profile and level, lesser or equal to the values of the level and
  profile declared in the SDP by the receiving agent.


### Declarative SDP Considerations

> **TODO**: Are theses rules correct for G-PCC and do we need considerations for
> other parameters?

When G-PCC over RTP is offered with SDP in a declarative style, as in Real Time
Streaming Protocol (RTSP) {{!RFC7826}} or Session Announcement Protocol (SAP)
{{!RFC2974}}, the following considerations are necessary.

* All parameters capable of indicating both stream properties and receiver
  capabilities are used to indicate only stream properties. In this case, the
  parameters profile and level declare only the values used by the stream, not
  the capabilities for receiving streams.
* A receiver of the SDP is required to support all parameters and values of the
  parameters provided; otherwise, the receiver MUST reject (RTSP) or not
  participate in (SAP) the session. It falls on the creator of the session to
  use values that are expected to be supported by the receiving application.

# Congestion Control

Congestion control for RTP SHALL be used in accordance with {{!RFC3550}}, and
with any applicable RTP profile: e.g., {{!RFC3551}}.  An additional requirement
if best-effort service is being used is users of this payload format MUST
monitor packet loss to ensure that the packet loss rate is within acceptable
parameters. Circuit Breakers {{!RFC8083}} is an update to RTP {{!RFC3550}} that
defines criteria for when one is required to stop sending RTP Packet Streams.
The circuit breakers is to be implemented and followed.

The bitrate can be dynamically adapted when real-time encoding is used. If the
packet payload is not encrypted and intermediate network elements have access to
the payload, they can select packets to drop based on the payload header.

Attribute decoding depends on geometry decoding and ADUs can become obsolete,
when the corresponding geometry data is dropped. Thus, intermediates SHOULD
prefer dropping ADUs first. Similarly, fragmentation units can only be
reconstructed if all fragments arrive. Thus, it is reasonable to drop either all
or none of the packets that are part of the same fragmentation unit.

# IANA Considerations

> **TODO**: Check if more registrations are necessary.

This memo requests that IANA registers a new media type as specified in
{{media-type}}. The media type is also requested to be added to the IANA
registry for "RTP Payload Format MIME types"
<http://www.iana.org/assignments/rtp-parameters>.

# Security Considerations {#security-considerations}

RTP packets using the payload format defined in this specification are subject
to the security considerations discussed in the RTP specification [RFC3550] ,
and in any applicable RTP profile such as RTP/AVP [RFC3551], RTP/AVPF [RFC4585],
RTP/SAVP [RFC3711], or RTP/ SAVPF [RFC5124].  However, as "Securing the RTP
Protocol Framework: Why RTP Does Not Mandate a Single Media Security Solution"
[RFC7202] discusses, it is not an RTP payload format's responsibility to discuss
or mandate what solutions are used to meet the basic security goals like
confidentiality, integrity, and source authenticity for RTP in general.  This
responsibility lays on anyone using RTP in an application.  They can find
guidance on available security mechanisms and important considerations in
"Options for Securing RTP Sessions" [RFC7201].  Applications SHOULD use one or
more appropriate strong security mechanisms.  The rest of this Security
Considerations section discusses the security impacting properties of the
payload format itself.

This RTP payload format and its media decoder do not exhibit any significant
non-uniformity in the receiver-side computational complexity for packet
processing, and thus are unlikely to pose a denial-of-service threat due to the
receipt of pathological data. Nor does the RTP payload format contain any active
content.

# RFC Editor Considerations

Note to RFC Editor: This section may be removed after carrying out all the
instructions of this section.

TODO: Consider

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
