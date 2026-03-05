---
title: "MoQ Media Interop"
abbrev: "moq-mi"
category: info

docname: draft-cenzano-moq-media-interop-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 4
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - Media over QUIC
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "afrind/draft-cenzano-media-interop"
  latest: "https://afrind.github.io/draft-cenzano-media-interop/draft-cenzano-moq-media-interop.html"

author:
 -
    fullname: Jordi Cenzano-Ferret
    organization: Meta
    email: jcenzano@meta.com

 -
    fullname: Alan Frindell
    organization: Meta
    email: afrind@meta.com

 -
    fullname: Paul Gregoire
    organization: Red5
    email: paul@red5.net

normative:

informative:

--- abstract

This protocol can be used to send and receive video and audio over Media over
QUIC Transport [MOQT], using LOC[loc] packaging.

--- middle

# Introduction

This protocol specifies a simple mechanism for sending media (video and audio)
over LOC[loc] for both live-streaming and video conference (VC) style use
cases.

moq-mi allows updating encoding parameters in the middle of a track (ex: frame
rate, resolution, codec, etc)

The protocol refers to [loc] to define the specific media wire format.

# Protocol Operation

## Track Names

The publisher selects a namespace of their choosing, and sends an ANNOUNCE
message for this namespace.

Within the publisher namespace the publisher will offer media tracks named as
`videoX` and `audioX` where X will be an integer starting at 0.

So in case the publisher issues 2 audio tracks and 1 video track, the track
names available will be `video0`, `audio0`, and `audio1`.

The subscriber will consider all of those tracks belonging to the same
namespace as part of the same synchronization group (timestamps aligned to the
same timeline).

## Mapping Tracks to MoQT Object Model

For the video track, the publisher begins a new group at the start of each IDR
(so object 0 will be always an IDR Keyframe), and each group contains a single
subgroup. Each object has the format described in {{object-format}}. This
applies to both H.264 and H.265 video tracks.

For HEVC/H.265, the same group/object mapping applies. A new group begins at
each IDR or CRA (Clean Random Access) picture, and object 0 within each group
is always a random access point.

For the audio track, the publisher begins a new group with each audio object,
and each group contains a single subgroup. Each object has the format described
in {{object-format}}.

TODO: Datagram forwarding preference could be used, but has problems if audio
frame does not fit in a single UDP payload.

## Timestamps

To avoid using fractional numbers and having to deal with rounding errors,
timestamps will be expressed with two integers:
- timestamp numerator (ex: PTS, DTS, duration)
- timebase

To convert a timestamp into seconds you just need to:
timestamp(s) = timestamp numerator / timebase

Example:

PTS = 11, timebase = 30

PTS(s) = 11/30 = 0.366666s

## Object Format

MoQ-MI uses MOQT extension headers to provide metadata that identifies and
augemts the media information found in the object payload.

### Media type header extension (header extension type = 0x0A)

It defines the media type inside object payload (see section IANA in MOQ TODO),
and it MUST be present in all objects

| Value | Media type |
|------:|:-----------|
| 0x0 | Video H264 in AVCC |
| 0x1 | Audio Opus bitsream |
| 0x2 | UTF-8 text |
| 0x3 | Audio AAC-LC in MPEG4 |
| 0x4 | Video H265 in HVCC |

### Video H264 in AVCC metadata header extension (header extension type = 0x15)

It provides video metadata useful to consume the video carried in the payload of
the object. The following table specifies the data inside this extesion header.

~~~
{
  Seq ID (i)
  PTS Timestamp (i)
  DTS Timestamp (i)
  Timebase (i)
  Duration (i)
  Wallclock (i)
}
~~~
{: #media-object-header-0b-video-h264-avcc format title="MOQT Media video h264data header extension"}

It MUST be present in all objects where "media type header extension" is equal
to "Video H264 in AVCC"(0x0)

#### Seq ID

Monotonically increasing counter for this media track

#### PTS Timestamp

Indicates PTS in timebase

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming)

#### DTS Timestamp

Not needed if B frames are NOT used, in that case should be same value as PTS.

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming)

#### Timebase

Units used in PTS, DTS, and duration.

#### Duration

Duration in timebase.
It will be 0 if not set

#### Wall Clock

EPOCH time in ms when this frame started being captured.
It will be 0 if not set

### Video H264 in AVCC extradata header extension (Header extension type = 0x0D)

Provides extradata needed to start decoding the video stream

It MUST be present in all object 0 (start of group) where "media type header
extension" is equal to "Video H264 in AVCC"(0x0) AND there has been an update on
the encoding paramets (or very start of the stream)

~~~
{
  Extradata (..)
}
~~~
{: #media-object-header-0d-video-h264-avcc format title="MOQT Media video h264 extradata header extension"}

#### Extradata

This will be `AVCDecoderConfigurationRecord` as described in
[ISO14496-15:2019] section 5.3.3.1, with field `lengthSizeMinusOne` = 3
(So length = 4). If any other size length is indicated (in
`AVCDecoderConfigurationRecord`) we should error with "Protocol violation"

### Audio Opus bitsream data header extension (Header extension type = 0x0F)

It provides audio metadata useful to consume the audio carried in the payload
of the object. Following table specifies the data inside this extesion header.

It MUST be present in all objects where "media type header extension" is equal
to "Audio Opus bitsream"(0x1)

~~~
{
  Seq ID (i)
  PTS Timestamp (i)
  Timebase (i)
  Sample Freq (i)
  Num Channels (i)
  Duration (i)
  Wall Clock (i)
}
~~~
{: #media-object-header-audio-opus format title="MOQT Media data audio Opus header extension"}

#### Seq Id

Monotonically increasing counter for this media track

#### PTS Timestamp

Indicates PTS in timebase

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming)

#### Timebase

Units used in PTS, DTS, and duration

#### Sample Freq

Sample frequency used in the original signal (before encoding)

#### Num Channels

Number of channels in the original signal (before encoding)

#### Duration

Duration in timebase.
It will be 0 if not set

#### Wallclock

EPOCH time in ms when this frame started being captured.
It will be 0 if not set

### UTF-8 Text header extension (Header extension type = 0x11)

~~~
{
  Seq ID (i)
}
~~~
{: #object-header-utf8-text format title="MOQT UTF-8 Text header extension"}

#### Seq Id

Monotonically increasing counter for this track

### Audio AAC-LC in MPEG4 bitstream data header extension (Header extension type = 0x13)

~~~
{
  Seq ID (i)
  PTS Timestamp (i)
  Timebase (i)
  Sample Freq (i)
  Num Channels (i)
  Duration (i)
  Wall Clock (i)
}
~~~
{: #media-object-header-audio-aaclcmpeg4 format title="MOQT Media audio AAC-LC MPEG4 object header"}

#### Seq Id

Monotonically increasing counter for this media track

#### PTS Timestamp

Indicates PTS in timebase

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming)

#### Timebase

Units used in PTS, DTS, and duration

#### Sample Freq

Sample frequency used in the original signal (before encoding)

#### Num Channels

Number of channels in the original signal (before encoding)

#### Duration

Duration in timebase.
It will be 0 if not set

#### Wallclock

EPOCH time in ms when this frame started being captured.
It will be 0 if not set

### Video H265 in HVCC metadata header extension (header extension type = 0x17)

It provides video metadata useful to consume the H.265/HEVC video carried in
the payload of the object. The following table specifies the data inside this
extension header.

~~~
{
  Seq ID (i)
  PTS Timestamp (i)
  DTS Timestamp (i)
  Timebase (i)
  Duration (i)
  Wallclock (i)
}
~~~
{: #media-object-header-video-h265-hvcc format title="MOQT Media video H265 in HVCC metadata header extension"}

It MUST be present in all objects where "media type header extension" is equal
to "Video H265 in HVCC"(0x4)

#### Seq ID

Monotonically increasing counter for this media track

#### PTS Timestamp

Indicates PTS in timebase

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming)

#### DTS Timestamp

Not needed if B frames are NOT used, in that case should be same value as PTS.

Note: HEVC supports B-frame prediction structures (including generalized
B-frames and temporal sub-layers). When B-frames are present, DTS will differ
from PTS and both values MUST be set correctly.

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming)

#### Timebase

Units used in PTS, DTS, and duration.

#### Duration

Duration in timebase.
It will be 0 if not set

#### Wall Clock

EPOCH time in ms when this frame started being captured.
It will be 0 if not set

### Video H265 in HVCC extradata header extension (Header extension type = 0x19)

Provides extradata (decoder configuration) needed to start decoding the
H.265/HEVC video stream.

It MUST be present in object 0 (start of group) where "media type header
extension" is equal to "Video H265 in HVCC"(0x4) AND there has been an update
on the encoding parameters (or very start of the stream).

~~~
{
  Extradata (..)
}
~~~
{: #media-object-header-video-h265-extradata format title="MOQT Media video H265 in HVCC extradata header extension"}

#### Extradata

This MUST be `HEVCDecoderConfigurationRecord` as described in [ISO14496-15]
section 8.3.3.1.2, with field `lengthSizeMinusOne` = 3 (So length = 4). If any
other size length is indicated (in `HEVCDecoderConfigurationRecord`) we should
error with "Protocol violation".

The `HEVCDecoderConfigurationRecord` contains the VPS, SPS, and PPS NAL units
required to initialize the HEVC decoder.

# Examples

## Example of first object for Video H264 in AVCC

### MOQT Object subgroup fields in case it carries Video H264 in AVCC, 1st frame sent

~~~
{
  0x00 (Object ID)(i),
  0x03 (Extension Count)(i),
  0x0A (Header type: Media type header type)(i)
  0x00 (header value: Media type)(i)

  0x15 (Header type: H264 in AVCC metadata)(i)
  0x0D (Header value length)(i)
  0x00 (Header value: Seq ID)(i)
  0x00 (Header value: PTS Timestamp)(i)
  0x00 (Header value: DTS Timestamp)(i)
  0x1E (Header value: Timebase)(i)
  0x01 (Header value: Duration)(i)
  0xC0, 0x00, 0x01, 0x95, 0x45, 0x6C, 0x8B, 0xFF (Header value: Wallclock)(i)

  0x0D (Header type: H264 in AVCC extradata)(i)
  Header value length (i)
  Header value: H264 in AVCC extradata (..)

  Object Payload Length (i),
  Object Payload bytes (..),
}
~~~

### MOQT Object subgroup fields in case it carries Video H264 in AVCC, 2nd or bigger frame with NO encoding settings update

~~~
{
  0x01 (Object ID)(i),
  0x02 (Extension Count)(i),
  0x0A (Header type: Media type header type)(i)
  0x00 (header value: Media type)(i)

  0x15 (Header type: H264 in AVCC metadata)(i)
  0x0D (Header value length)(i)
  0x01 (Header value: Seq ID)(i)
  0x00 (Header value: PTS Timestamp)(i)
  0x00 (Header value: DTS Timestamp)(i)
  0x1E (Header value: Timebase)(i)
  0x01 (Header value: Duration)(i)
  0xC0, 0x00, 0x01, 0x95, 0x45, 0x6C, 0x3B, 0xE0 (Header value: Wallclock)(i)

  Object Payload Length (i),
  Object Payload bytes (..),
}
~~~

### MOQT Object DATAGRAM fields in case it carries Audio AAC-LC in MPEG4

~~~
{
  0x00 (Track Alias)(i),
  0x00 (Group ID)(i),
  0x00 (Object ID)(i),
  0x00 (Publisher Priority)(8),
  0x02 (Extension Count)(i),
  0x0A (Header type: Media type header type)(i)
  0x03 (header value: Media type)(i)

  0x13 (Header type: Audio AAC-LC in MPEG4)(i)
  0x15 (Header value length)(i)
  0x00 (Header value: Seq ID)(i)
  0x00 (Header value: PTS Timestamp)(i)
  0x80, 0x00, 0xBB, 0x80 (Header value: Timebase)(i)
  0x80, 0x00, 0xBB, 0x80 (Header value: Sample freq)(i)
  0x02 (Header value: Num channels)(i)
  0x44, 0x00 (Header value: Duration)(i)
  0xC0, 0x00, 0x01, 0x95, 0x45, 0x6C, 0x3B, 0xE0 (Header value: Wallclock)(i)

  Object Payload Length (i),
  Object Payload bytes (..),
}
~~~

## Example of first object for Video H265 in HVCC

### MOQT Object subgroup fields in case it carries Video H265 in HVCC, 1st frame sent

~~~
{
  0x00 (Object ID)(i),
  0x03 (Extension Count)(i),
  0x0A (Header type: Media type header type)(i)
  0x04 (header value: Media type = Video H265 in HVCC)(i)

  0x17 (Header type: H265 in HVCC metadata)(i)
  0x0D (Header value length)(i)
  0x00 (Header value: Seq ID)(i)
  0x00 (Header value: PTS Timestamp)(i)
  0x00 (Header value: DTS Timestamp)(i)
  0x1E (Header value: Timebase)(i)
  0x01 (Header value: Duration)(i)
  0xC0, 0x00, 0x01, 0x95, 0x45, 0x6C, 0x8B, 0xFF (Header value: Wallclock)(i)

  0x19 (Header type: H265 in HVCC extradata)(i)
  Header value length (i)
  Header value: H265 in HVCC extradata (HEVCDecoderConfigurationRecord) (..)

  Object Payload Length (i),
  Object Payload bytes (..),
}
~~~

### MOQT Object subgroup fields in case it carries Video H265 in HVCC, 2nd or bigger frame with NO encoding settings update

~~~
{
  0x01 (Object ID)(i),
  0x02 (Extension Count)(i),
  0x0A (Header type: Media type header type)(i)
  0x04 (header value: Media type = Video H265 in HVCC)(i)

  0x17 (Header type: H265 in HVCC metadata)(i)
  0x0D (Header value length)(i)
  0x01 (Header value: Seq ID)(i)
  0x00 (Header value: PTS Timestamp)(i)
  0x00 (Header value: DTS Timestamp)(i)
  0x1E (Header value: Timebase)(i)
  0x01 (Header value: Duration)(i)
  0xC0, 0x00, 0x01, 0x95, 0x45, 0x6C, 0x3B, 0xE0 (Header value: Wallclock)(i)

  Object Payload Length (i),
  Object Payload bytes (..),
}
~~~

# Payloads

TODO: This sections needs to be updated with links to LOC

## For object header media type = Video H264 in AVCC (0x00)

Payload MUST be H264 with bitstream AVC1 format as described in
[ISO14496-15:2019] section 5.3. Using 4 bytes size field length.

## For object header media type = Audio Opus bitsream (0x01)

Payload MUST be Opus packets, as described in {{!RFC6716}} - section 3

## For object header media type = UTF-8 text (0x02)

Payload MUST be text bytes in UTF-8, as described in {{!RFC3629}}

## For object header media type = Audio AAC-LC in MPEG4 (0x03)

Payload MUST be AAC frame (syntax element `raw_data_block()`), as described in
section 4.4.2.1 of [ISO14496-3:2009].

## For object header media type = Video H265 in HVCC (0x04)

Payload MUST be H.265/HEVC with bitstream HVC1 format as described in
[ISO14496-15] section 8.3. Using 4 bytes size field length, as indicated by
`lengthSizeMinusOne = 3` in the `HEVCDecoderConfigurationRecord`.

Each payload contains one or more HEVC NAL units prefixed with a 4-byte length
field. The NAL unit types follow [ISO23008-2]:

- IDR_W_RADL (19), IDR_N_LP (20): Instantaneous Decode Refresh
- CRA_NUT (21): Clean Random Access
- TRAIL_N (0), TRAIL_R (1): Trailing pictures
- TSA_N (2), TSA_R (3): Temporal Sub-layer Access
- STSA_N (4), STSA_R (5): Step-wise Temporal Sub-layer Access
- BLA_W_LP (16), BLA_W_RADL (17), BLA_N_LP (18): Broken Link Access

# References

[ISO14496-15:2019] "Carriage of network abstraction layer (NAL) unit structured
video in the ISO base media file format", ISO ISO14496-15:2019, International
Organization for Standardization, October, 2022.
Note: Section 5.3 defines AVC (H.264) sample format and
AVCDecoderConfigurationRecord. Section 8.3 defines HEVC (H.265) sample format
and HEVCDecoderConfigurationRecord.

[ISO14496-3:2009] "Information technology - Coding of audio-visual objects",
ISO ISO14496-3:2009, International Organization for Standardization, September,
2009.

[ISO23008-2:2020] "Information technology - High efficiency coding and media
delivery in heterogeneous environments - Part 2: High efficiency video coding",
ISO/IEC 23008-2:2020 (also published as ITU-T H.265), International
Organization for Standardization, June, 2020.
Note: Defines the HEVC video coding standard including NAL unit types, picture
types (IDR, CRA, BLA, TSA, STSA), and temporal sub-layer structures referenced
in Section 4.5.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

# Extension Summary
{:numbered="false"}

The following table summarizes all header extension types:

| Extension ID | Name | Value Format |
|:-------------|:-----|:-------------|
| 0x0A (even) | Media Type | Varint |
| 0x0D (odd) | H264 Extradata (AVCC) | Byte array (length) |
| 0x0F (odd) | Opus Audio Data | Byte array (length) |
| 0x11 (odd) | UTF-8 Text | Byte array (length) |
| 0x13 (odd) | AAC-LC Audio Data | Byte array (length) |
| 0x15 (odd) | H264 Metadata (AVCC) | Byte array (length) |
| 0x17 (odd) | H265 Metadata (HVCC) | Byte array (length) |
| 0x19 (odd) | H265 Extradata (HVCC) | Byte array (length) |

Even extension IDs carry a varint value directly. Odd extension IDs carry a
varint length followed by that many bytes.
