---
title: "MoQ Media Interop"
abbrev: "moq-mi"
category: info

docname: draft-cenzano-moq-media-interop-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
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

normative:

informative:

--- abstract

This protocol can be used to send and receive video and audio over Media over
QUIC Transport [MOQT], using LOC[loc] packaging.

--- middle

# Introduction

This protocol specifies a simple mechanism for sending media (video and audio)
over LOC[loc] for both live-streaming and video conference (VC) style use cases.

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
subgroup.  Each object has the format described in {{object-format}}.

For the audio track, the publisher begins a new group with each audio object,
and each group contains a single subgroup.  Each object has the format described
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

MoQ-MI uses MOQT extension headers to provide metadata that identifies
and augemts the media information found in the object payload.


### Media type header extension (header extension type = 0x0A)

It defines the media type inside object payload (see section IANA in MOQ TODO),
and it MUST be present in all objects

|-------|--------------------------------------|
| Value | Media type                           |
|------:|:-------------------------------------|
| 0x0   | Video H264 in AVCC                   |
|-------|--------------------------------------|
| 0x1   | Audio Opus bitsream                  |
|-------|--------------------------------------|
| 0x2   | UTF-8 text                           |
|-------|--------------------------------------|
| 0x3   | Audio AAC-LC in MPEG4                |
|-------|--------------------------------------|


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

This will be  `AVCDecoderConfigurationRecord` as described in
[ISO14496-15:2019] section 5.3.3.1, with field `lengthSizeMinusOne` = 3
(So length = 4). If any other size length is indicated
(in `AVCDecoderConfigurationRecord`) we should error with “Protocol violation”


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

# Payloads

TODO: This sections needs to be updated with links to LOC

## For object header media type = Video H264 in AVCC (0x00)

Payload MUST be H264 with bitstream AVC1 format as described in [ISO14496-15:2019] section 5.3.
Using 4 bytes size field length.


## For object header media type = Audio Opus bitsream (0x01)

Payload MUST be Opus packets, as described in {{!RFC6716}} - section 3


## For object header media type = UTF-8 text (0x02)

Payload MUST be text bytes in UTF-8, as described in {{!RFC3629}}


## For object header media type = Audio AAC-LC in MPEG4 (0x03)

Payload MUST be AAC frame (syntax element `raw_data_block()`), as described in section 4.4.2.1 of [ISO14496-3:2009].


# References

[ISO14496-15:2019] "Carriage of network abstraction layer (NAL) unit
structured video in the ISO base media file format", ISO ISO14496-15:2019,
International Organization for Standardization, October, 2022.

[ISO14496-3:2009] "Information technology — Coding of audio-visual objects",
ISO ISO14496-3:2009, International Organization for Standardization, September, 2009.

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
