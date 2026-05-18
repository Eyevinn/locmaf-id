---
title: "Low Overhead CMAF for Media over QUIC (LOCMAF)"
abbrev: "LOCMAF"
docname: draft-einarsson-moq-locmaf-latest
category: info
submissiontype: IETF
number:
date:
v: 3
area: ART
workgroup: "Media Over QUIC"
keyword:
 - moq
 - cmaf
 - live streaming
 - low latency

venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "Eyevinn/locmaf-id"
  latest: "https://Eyevinn.github.io/locmaf-id/draft-einarsson-moq-locmaf.html"

author:
 -
    fullname: Torbjörn Einarsson
    organization: Eyevinn Technology
    email: torbjorn.einarsson@eyevinn.se
 -
    fullname: Hugo Björs
    organization: Eyevinn Technology
    email: hugo.bjors@eyevinn.se

normative:
  MOQT: I-D.ietf-moq-transport
  CMSF: I-D.ietf-moq-cmsf
  RFC2119:
  RFC8174:

informative:
  CMAF:
    title: "Information technology — Multimedia application format (MPEG-A) — Part 19: Common media application format (CMAF) for segmented media"
    seriesinfo:
      ISO/IEC: 23000-19
    date: 2020
  ISOBMFF:
    title: "Information technology — Coding of audio-visual objects — Part 12: ISO base media file format"
    seriesinfo:
      ISO/IEC: 14496-12
    date: 2022
  CENC:
    title: "Information technology — MPEG systems technologies — Part 7: Common encryption in ISO base media file format files"
    seriesinfo:
      ISO/IEC: 23001-7
    date: 2023
  LOC: I-D.cenc-moq-loc
  COMPRESSED-MP4: I-D.curley-moq-compressed-mp4
  MOQLIVEMOCK:
    target: https://github.com/Eyevinn/moqlivemock
    title: "moqlivemock — Reference LOCMAF server and tooling"
  LOCMAF-SITE:
    target: https://locmaf.dev
    title: "LOCMAF — Low Overhead CMAF for MoQ"


--- abstract

This document specifies LOCMAF (Low Overhead CMAF for MoQ), a wire
format for streaming low-latency CMAF media over the MoQ Transport
protocol (MOQT) with per-object overhead comparable to the Low
Overhead Container (LOC). LOCMAF compresses the `moof` boxes of
consecutive CMAF chunks within a MoQ group by transmitting one full
`moof` per group followed by compact delta encodings, while leaving
sample data (`mdat`) untouched. The receiver reconstructs CMAF
chunks that are semantically equivalent to the sender input,
preserving the encryption metadata required by CMAF DRM
(Common Encryption) pipelines.


--- middle

# Introduction

CMAF {{CMAF}} chunk headers are typically over 100 bytes, while the
codec frames they describe may be just a few hundred bytes at low
latency. Streaming CMAF directly over MOQT {{MOQT}} therefore incurs
a per-object overhead that the Low Overhead Container (LOC)
{{LOC}} avoids by carrying raw codec frames. LOC, however, cannot
transport the per-sample CENC {{CENC}} metadata needed for browser
EME / CDM decryption of DRM-protected live streams.

LOCMAF closes this gap. It exploits the observation that consecutive
`moof` boxes within a single CMAF segment are nearly identical:
the first `moof` of a MoQ group is sent in full, subsequent `moof`
boxes are sent as compact deltas, and `mdat` payloads are passed
through unchanged. The receiver reconstructs full CMAF chunks that
are byte-compatible enough to feed unmodified MSE / EME pipelines.

This document specifies the LOCMAF object framing, the full and
delta `moof` encodings, the CMSF {{CMSF}} catalog signalling
(`locmafVersion`), and the receiver reconstruction algorithm.

A reference implementation is available {{MOQLIVEMOCK}}. Worked
examples and diagrams are published at {{LOCMAF-SITE}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

CMAF chunk:
: A `moof` + `mdat` pair as defined in {{CMAF}}.

CMAF segment:
: A sequence of CMAF chunks starting at a random access point.

MoQ group, MoQ object:
: As defined in {{MOQT}}.

Full `moof`:
: The first CMAF chunk header in a MoQ group, sent verbatim
  (with light normalisation, see {{full-moof}}).

Delta `moof`:
: A subsequent CMAF chunk header in the same group, encoded as a
  difference against an in-group reference, see {{delta-moof}}.

# MoQ Group / Object Mapping {#mapping}

LOCMAF assumes the following mapping from CMAF to MOQT:

- One MoQ group per CMAF segment. Group boundaries align with
  random access points.
- One MoQ object per CMAF chunk. Each object carries the (possibly
  compressed) `moof` followed by the unmodified `mdat`.
- Audio MoQ groups have the same duration as the video MoQ groups
  with which they will be muxed, to enable joint tune-in.

Within a MoQ group, all objects MUST be delivered in order.
Across groups, no ordering assumption is made: every group is
independently decodable.

# CMSF Catalog Signalling

A track that carries LOCMAF-encoded chunks MUST advertise a
`locmafVersion` property in its CMSF {{CMSF}} catalog entry.
This document defines version `"0.1"`.

\[TODO: full catalog property definition, IANA registry pointer.]

# Object Framing

\[TODO: object header layout, including `header_id`,
`properties_length`, and the framing rules for full vs. delta
`moof`. Reference the diagrams under `assets/diagrams/` in the
companion site repo.]

# Full `moof` Encoding {#full-moof}

\[TODO: which boxes are required, normalisation rules
(single-sample size inference, implicit `trex` defaults),
and the byte budget.]

# Delta `moof` Encoding {#delta-moof}

\[TODO: which fields are carried in a delta, how the reference
`moof` is selected within a group, and the sample-size table
handling.]

# CMAF Init Segment Compression

\[TODO: optional use of LOCMAF for compressing the CMAF init
segment carried as a track's MoQ catalog payload. Compare against
{{COMPRESSED-MP4}}.]

# Receiver Reconstruction

\[TODO: step-by-step algorithm for rebuilding a byte-equivalent
CMAF chunk stream that can be fed to an MSE / EME pipeline.]

# DRM with LOCMAF

\[TODO: how `senc`, subsample maps, and `tenc` defaults survive
the LOCMAF round-trip; pipeline diagram pointer.]

# Security Considerations

LOCMAF is a compression layer over CMAF media and does not introduce
new authentication or confidentiality mechanisms. It is intended to
be used over MOQT {{MOQT}}, which inherits QUIC's transport
security. Per-sample encryption metadata defined by {{CENC}} is
preserved through the LOCMAF round-trip; LOCMAF neither weakens nor
strengthens the underlying DRM scheme.

A receiver MUST validate that reconstructed `moof` boxes are
well-formed before passing them to a media pipeline, since malformed
deltas could otherwise be used to construct ISOBMFF {{ISOBMFF}}
boxes with inconsistent field lengths.

\[TODO: detail bounds checks, replay considerations within a group.]

# IANA Considerations

This document requests IANA to register the CMSF catalog property
`locmafVersion` in the \[TODO: name of CMSF catalog property
registry] registry, with the value `"0.1"` defined by this document.

\[TODO: complete registration template.]


--- back

# Acknowledgments
{:numbered="false"}

LOCMAF was developed as part of the Master Thesis work of Hugo
Björs at Eyevinn Technology, supervised by Torbjörn Einarsson.
The authors thank the Media over QUIC working group, in particular
the authors of {{LOC}} and {{COMPRESSED-MP4}}, for the prior art
this work builds on.
