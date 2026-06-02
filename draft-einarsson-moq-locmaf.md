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
    organization: KTH
    email: hugobjoers@gmail.com

normative:
  MOQT: I-D.ietf-moq-transport
  CMSF: I-D.ietf-moq-cmsf
  MSF: I-D.ietf-moq-msf

informative:
  CMAF:
    title: "Information technology — Multimedia application format (MPEG-A) — Part 19: Common media application format (CMAF) for segmented media"
    seriesinfo:
      ISO/IEC: 23000-19:2023
    date: 2023
  ISOBMFF:
    title: "Information technology — Coding of audio-visual objects — Part 12: ISO base media file format"
    seriesinfo:
      ISO/IEC: 14496-12:2022
    date: 2022
  CENC:
    title: "Information technology — MPEG systems technologies — Part 7: Common encryption in ISO base media file format files"
    seriesinfo:
      ISO/IEC: 23001-7:2024
    date: 2024-08
  DASH:
    title: "Information technology — Dynamic adaptive streaming over HTTP (DASH) — Part 1: Media presentation description and segment formats"
    seriesinfo:
      ISO/IEC: 23009-1:2022
    date: 2022
  LOC: I-D.ietf-moq-loc
  COMPRESSED-MP4: I-D.lcurley-compressed-mp4
  DASH-IF-INGEST:
    target: https://dashif.org/Ingest/
    title: "DASH-IF Live Media Ingest Protocol"
  MOQLIVEMOCK:
    target: https://github.com/Eyevinn/moqlivemock
    title: "moqlivemock — Reference LOCMAF server and tooling"
  LOCMAF-SITE:
    target: https://locmaf.dev
    title: "LOCMAF — Low Overhead CMAF for MoQ"


--- abstract

This document specifies LOCMAF (Low Overhead CMAF for Media over
QUIC), a compact wire format for streaming low-latency CMAF media
over the MoQ Transport protocol (MOQT) with per-object overhead
comparable to the Low Overhead Container (LOC). LOCMAF carries the
CMAF chunk head — `styp` (segment type), `prft` (producer
reference time), any number of `emsg` (event message) boxes, and
the `moof` (movie fragment) — as a small set of tagged fields
inside one of
two LOCMAF object kinds, while leaving the sample
data (`mdat`) untouched. The first object of each MOQT group carries
a full reference; subsequent objects in the same group carry only
the differences. The receiver reconstructs CMAF chunks that are
semantically equivalent to the sender input, including
encryption metadata required by CMAF DRM (Common Encryption)
pipelines.


--- middle

# Introduction

CMAF {{CMAF}} chunk headers are typically over 100 bytes, while the
codec frames they describe may be only a few hundred bytes at low
latency. Streaming CMAF directly over MoQ Transport {{MOQT}}
therefore incurs a per-object overhead that the Low Overhead
Container (LOC) {{LOC}} avoids by carrying raw codec frames. LOC,
however, cannot transport the per-sample CENC {{CENC}} metadata
needed for browser EME / CDM decryption of DRM-protected live
streams, nor the `prft` (Producer Reference Time) and `emsg` (DASH
Event Message) boxes that CMAF carries alongside the `moof`.

LOCMAF closes this gap. It exploits the observation that consecutive
CMAF chunk heads within a single CMAF segment are nearly identical:
the first chunk of a MOQT group is sent in full, subsequent chunks
are sent as compact deltas against the previous chunk in the same
group, and `mdat` payloads are passed through unchanged. The
receiver reconstructs full CMAF chunks that are byte-compatible
enough to feed unmodified MSE / EME pipelines.

This document specifies the LOCMAF object framing, the full and
delta chunk encodings, the CMSF {{CMSF}} catalog signalling, the
receiver reconstruction algorithm, and the DRM box round-trip.

## Relationship to prior work

The Compressed MP4 draft {{COMPRESSED-MP4}} addresses the general
problem of compressing ISO BMFF box payloads, including CMAF
initialisation segments. LOCMAF takes a complementary, narrower
approach:

- LOCMAF does **not** compress the CMAF Header
  (initialisation segment). The CMAF Header is carried verbatim in
  the catalog (see {{cmaf-header}}). Init compression is a
  one-time-per-track cost; LOCMAF's wire-byte target is the
  per-chunk overhead, which an init codec cannot reduce. Carrying
  the CMAF Header verbatim has a second, deployment-driven
  benefit: a `locmaf` packaging track uses the **same MSF
  initialisation-data mechanism** as a `cmaf` packaging track,
  and when both wrap the same source they MAY refer to the same
  init entry (see {{cmaf-header}}). Publishers can therefore
  introduce LOCMAF as a more efficient wire format for clients
  that support it without duplicating the initialisation bytes
  for legacy CMAF clients — both audiences consume the same init
  from the same catalog entry, and the publisher only adds the
  LOCMAF-encoded media track alongside the CMAF one. Where
  generic init compression is desired, {{COMPRESSED-MP4}} or a
  generic transport-layer compression layer can be applied
  independently of LOCMAF.
- LOCMAF's goal for the per-chunk path is **functionally
  equivalent reconstruction**, not byte-exact reconstruction. The
  reconstructed CMAF chunk MUST carry the same samples, sample
  metadata, and CENC metadata as the source, but byte-level
  details that do not affect a CMAF reader are not preserved. For
  example, the relative order of generated `saio` / `saiz` boxes
  inside the reconstructed `traf` is implementation-defined, the
  `trun.tr_flags` packing is canonicalised, and `tfhd` defaults
  that match `trex` are silently re-supplied on reconstruction. A
  receiver MUST NOT rely on byte-level identity with the source
  CMAF stream.

A reference implementation is available {{MOQLIVEMOCK}}. Worked
examples and diagrams are published at {{LOCMAF-SITE}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

CMAF chunk:
: One `moof` + one `mdat` pair, optionally preceded by at most one
  `styp`, at most one `prft`, and zero or more `emsg` boxes, as
  defined in {{CMAF}} §7.3.3.2. The smallest CMAF addressable
  media object.

CMAF fragment:
: One or more CMAF chunks whose first chunk starts at a Stream
  Access Point ({{CMAF}} §7.3.2.2). A fragment is logically a
  single `MovieFragmentBox` worth of samples; in "chunked" CMAF the
  samples are split across multiple smaller `moof` + `mdat` pairs.

CMAF segment:
: One or more CMAF fragments in decode order ({{CMAF}} §7.3.2.4).
  The segment is the typical unit of HTTP delivery in DASH and
  HLS-fMP4; in LOCMAF the segment corresponds to one MOQT group.

CMAF Header:
: The `ftyp` + `moov` pair that initialises a CMAF track. Also
  called an *initialisation segment* in DASH parlance and carried
  as `initData` in MSF {{MSF}} / CMSF {{CMSF}} catalogs.

MOQT group, MOQT object:
: As defined in {{MOQT}}.

LOCMAF object:
: A MOQT object whose payload begins with one of the top-level
  header IDs defined in {{object-framing}}.

Full LOCMAF chunk:
: A LOCMAF object whose top-level header ID is `LocmafFullHeader`.
  It carries an absolute encoding of the CMAF chunk head and serves
  as the in-group reference for subsequent delta objects. See
  {{full-chunk}}.

Delta LOCMAF chunk:
: A LOCMAF object whose top-level header ID is `LocmafDeltaHeader`.
  It encodes differences against the most recently received full
  LOCMAF chunk in the same MOQT group. See {{delta-chunk}}.

BMDT:
: Abbreviation for `tfdt.baseMediaDecodeTime` ({{ISOBMFF}}).

# MOQT Group / Object Mapping {#mapping}

LOCMAF assumes the following mapping from CMAF to MOQT:

- One MOQT group per CMAF segment. Group boundaries align with
  random access points.
- One MOQT object per CMAF chunk. Each MOQT object is a LOCMAF
  object carrying the (full or delta) chunk head followed by the
  unmodified `mdat` payload.
- Audio MOQT groups typically have the same duration as the video MOQT groups
  with which they will be muxed, to enable joint tune-in.
- Sparse tracks, such as subtitle, or event/metadata tracks,
  are more likely to have groups that are not aligned with video.

Per {{MOQT}}, objects within a MOQT group are delivered in order
and groups are independently decodable. LOCMAF relies on both
properties: delta chunks reference the preceding chunk in the same
group (see {{delta-chunk}}), and each group MUST begin with a
`LocmafFullHeader` so a subscriber tuning in at a group boundary
has a complete reference (see {{dispatch}}).

# CMSF Catalog Signalling {#catalog}

A track that carries LOCMAF-encoded chunks MUST advertise:

- `packaging` equal to `"locmaf"`.
- `locmafVersion` equal to `"0.2"` for this version of the
  specification.

`locmafVersion` is present only when `packaging == "locmaf"` and is
omitted otherwise. Receivers MUST compare against their highest
supported version and SHOULD refuse the track when the encoder
advertises a version they do not implement.

The CMAF Header for a LOCMAF track is carried by the same MSF
{{MSF}} mechanism that carries the CMAF Header for a plain `cmaf`
packaging track. LOCMAF does not define its own init-carriage
shape, nor add `locmaf`-specific catalog fields beyond
`locmafVersion`. Whatever mechanism MSF specifies for `cmaf`
init data — inline `initData` today, or whichever indirection
form MSF adopts (e.g. a root-level init-data list referenced from
the track entry) — applies unchanged to `locmaf` tracks.

A consequence: when a `cmaf` packaging track and a `locmaf`
packaging track wrap the same source, they MAY refer to the same
MSF init-data entry. The wrapped media is identical at the
sample level and the CMAF Header bytes are identical at the byte
level (see {{cmaf-header}}); only the per-chunk wire encoding
differs.

\[TODO: IANA registry pointer for `locmafVersion`.]

# CMAF Header Delivery {#cmaf-header}

The CMAF Header for a LOCMAF track is byte-identical to the CMAF
Header a plain `cmaf` packaging track of the same source would
carry — `ftyp` followed by `moov`, followed by any optional
supplemental boxes (`pssh`, `mvex` with `trex`, etc.). It is
delivered uncompressed via the catalog using the same MSF
mechanism as for `cmaf` packaging. There is no LOCMAF-specific
CMAF Header carrier.

The `moov` in the CMAF Header MUST contain exactly one `trak` box
(see {{scope}}).

A LOCMAF receiver:

1. Resolves the track's MSF init-data reference (whichever form
   the catalog uses) to the base64-encoded CMAF Header bytes.
2. Base64-decodes the CMAF Header bytes.
3. Feeds the bytes to its MSE / decoder pipeline exactly as it
   would for a plain CMAF track.
4. Begins receiving LOCMAF-encoded media objects on the subscribed
   track and reconstructs each CMAF chunk from the LOCMAF payload.
5. Extracts the parameters required to regenerate CMAF chunks —
   `track_ID`, media timescale, `trex` defaults, and any track-
   encryption information (`tenc` defaults, default KID, default
   IV, scheme type, pattern parameters) — from the decoded CMAF
   Header. These values seed the reconstruction state used in
   step 4.

Compression of the catalog itself is out of scope for LOCMAF and
handled at the MOQT / MSF {{MSF}} layer.

# Scope and Publisher Requirements {#scope}

LOCMAF v0.2 targets the low-latency CMAF case: short CMAF fragments
composed of small CMAF chunks (often one sample per chunk),
optionally carrying CENC encryption metadata. To keep the wire
format minimal, the following constraints apply.

## Mandatory preconditions

A LOCMAF publisher MUST ensure that:

1. **Single `trak` per `moov`.** The CMAF Header contains exactly
   one `trak` box. Multi-track ISO BMFF files MUST be demuxed
   before LOCMAF encoding.
2. **No KID change within a CMAF chunk.** Key-identifier
   transitions MUST align with fragment (and therefore chunk)
   boundaries. This removes the need for `sgpd` / `sbgp` in the
   wire format.
3. **Restricted `sample_flags` populations.** Per-sample, default,
   and first-sample `sample_flags` MUST populate only `is_leading`,
   `sample_depends_on`, `sample_is_depended_on`, and
   `sample_is_non_sync_sample`; the fields `sample_has_redundancy`,
   `sample_padding_value`, and `sample_degradation_priority` MUST
   be zero in the source. See {{sample-flags}}.
4. **`emsg` version 1 only.** Any `emsg` boxes in the source MUST
   be version 1 per CMAF §7.4.5. See {{emsg}}.

If a source violates any of these, the publisher MUST NOT use
LOCMAF packaging for that track and MUST instead use plain CMAF or
an alternative packaging (e.g. an MSF `eventtimeline` companion
track for events). LOCMAF and plain CMAF tracks MAY coexist in the
same catalog under the same namespace.

## Recommended source properties

The following are recommendations whose violation costs wire bytes
but does not break LOCMAF:

1. **Commensurate media timescales.** Choose a timescale so every
   frame has an exact integer duration (e.g. 48 000 for 48 kHz AAC,
   60 000 for 60000/1001 fps video).
2. **Stable `trex` defaults.** Keeping `trex` consistent across the
   stream maximises what can be omitted from each chunk header.

## `tfdt.baseMediaDecodeTime` contiguity

CMAF (§7.5.18) requires that each fragment's BMDT equal the
previous fragment's BMDT plus the sum of its sample durations. The
delta-chunk BMDT derivation defined in {{delta-chunk}} relies on
this property. Re-anchoring is signalled in-band by emitting an
absolute BMDT override (see {{delta-chunk}}).

## Optional encoder modes

A LOCMAF encoder MAY operate in **strict `cmf2` mode**, in which it
always emits the four `tfhd` defaults (sample duration, size,
flags, sample-description index) in the full chunk header even
when they match `trex`. This costs ~6 B per group but produces
reconstructed CMAF chunks that satisfy CMAF §7.7.3 fragment self-
decodability (each chunk is a single-chunk fragment in the LOCMAF
mapping). Strict `cmf2` mode is signalled out-of-band; it does not
affect wire compatibility between encoders and decoders.

# Object Framing {#object-framing}

## Top-level header IDs

LOCMAF defines two top-level header IDs:

| ID | Symbol               | Object kind                              |
|----|----------------------|------------------------------------------|
| 23 | `LocmafFullHeader`   | Full LOCMAF chunk (see {{full-chunk}})   |
| 25 | `LocmafDeltaHeader`  | Delta LOCMAF chunk (see {{delta-chunk}}) |

Receivers MUST skip (and SHOULD log) unrecognised `header_id`
values rather than abort. The MOQT object length terminates the
unknown object cleanly.

Future extensions adding new top-level object kinds use any
unassigned ID, allocated via the IANA registry defined in
{{iana}}.

## Object layout

~~~
+-----------------------------+
| header_id        (varint)   |   top-level object kind
+-----------------------------+
| properties_length (varint)  |   length of the properties block in bytes
+-----------------------------+
| properties      (variable)  |   sequence of (field_id, value) tuples
+-----------------------------+
| mdat raw payload (rest)     |   length = MOQT-object-len - (above)
+-----------------------------+
~~~

`header_id` and `properties_length` are variable-length integers
using the encoding defined by MoQ Transport {{MOQT}} for the
session's MOQT version.

The mdat payload is the contents of the CMAF `mdat` box — the
sample data, without the surrounding 8-byte `size + 'mdat'` box
header. The receiver reconstructs a standard `mdat` box by wrapping
these bytes in an 8-byte ISO BMFF header.

For event-only tracks (see {{event-only}}) the mdat payload MAY be
zero bytes; the receiver reconstructs an empty `mdat` box (8-byte
header only).

## Property encoding (parity rule)

The properties block is a flat sequence of `(field_id, value)`
tuples. Field IDs are MOQT varints. The value encoding is
determined by the parity of the ID:

- **Even ID:** scalar varint. The value is a single MOQT varint.
  No length prefix. In delta chunks, the encoded value is a
  zigzag varint (see {{zigzag}}) of the signed delta against the
  in-group reference; in full chunks it is an absolute unsigned
  MOQT varint.
- **Odd ID:** length-prefixed bytes. The tuple is `field_id |
  value_length | value_bytes`. The interpretation of the bytes is
  per-field; varint-list fields concatenate elements (each element
  is a zigzag varint (see {{zigzag}}) in delta context, an
  absolute MOQT varint in full context), raw-bytes fields carry
  opaque content.

Field IDs MAY appear in any order; receivers MUST tolerate any
ordering. Encoders SHOULD emit IDs in ascending order to produce
deterministic wire bytes.

## Zigzag varint encoding {#zigzag}

A *zigzag varint* is a signed integer encoded as an unsigned MOQT
varint by interleaving non-negative and negative values so that
small-magnitude values of either sign occupy small unsigned
values, and thus the shortest varint forms.

For a signed 64-bit integer `n`, the mapping to its unsigned
zigzag representation `z` is:

~~~
encode:  z = (n << 1) ^ (n >> 63)              ; arithmetic right shift
                                                ; equivalently:
                                                ;   n >= 0:  z = 2 * n
                                                ;   n <  0:  z = -2 * n - 1

decode:  n = (z >> 1) ^ -(z & 1)               ; equivalently:
                                                ;   z even:  n =  z / 2
                                                ;   z odd:   n = -(z + 1) / 2
~~~

The first few mappings: 0↔0, -1↔1, 1↔2, -2↔3, 2↔4, -3↔5, 3↔6, ….

The encoded `z` is then serialised as an unsigned MOQT varint
({{MOQT}}); decoders read the MOQT varint and apply the decode
rule above.

This is the same zigzag mapping used by, e.g., Protocol Buffers;
the description is included here for self-containment of the
LOCMAF wire format.

LOCMAF uses zigzag varints wherever a signed delta against the
in-group reference is written, namely in scalar even-ID fields
(see above) and per-element in varint-list odd-ID fields. Absolute
values in `LocmafFullHeader` are encoded as plain unsigned MOQT
varints, not zigzag.

## Full vs delta dispatch {#dispatch}

The full-vs-delta distinction is signalled exclusively by the
top-level `header_id`, never by the MOQT object position within a
group.

1. The first object of every MOQT group MUST be a
   `LocmafFullHeader`.
2. The encoder MAY emit a `LocmafFullHeader` at any object
   position within a group, not only at object index 0. A
   mid-group full chunk re-anchors the in-group reference for
   subsequent delta chunks.
3. After receiving a `LocmafFullHeader`, the decoder MUST discard
   its in-group delta state and treat the new full chunk as the
   reference for any following `LocmafDeltaHeader` objects in the
   group.
4. The receiver MUST dispatch on `header_id` alone. It MUST NOT
   infer "full" from object index 0 or "delta" from object index >
   0.

# Field Reference {#field-ref}

Field IDs are organised in blocks by source box:

| Range       | Block                                |
|-------------|--------------------------------------|
| 1–16        | moof fields                          |
| 18, 20, 22, 24 | prft fields                       |
| 23          | styp field                           |
| 25          | emsg list                            |
| 27          | delta deletion marker                |
| 29+         | reserved for future extension        |

## moof fields {#moof-fields}

Fields whose source is the `moof` (and its enclosed `traf`):

| ID | Symbol                             | Source ISO BMFF field                            | Kind                |
|----|------------------------------------|--------------------------------------------------|---------------------|
|  1 | `moofSampleSizes`                  | `trun.samples[i].sample_size`                    | varint list         |
|  2 | `moofSampleDescriptionIndex`       | `tfhd.sample_description_index`                  | scalar              |
|  3 | `moofSampleDurations`              | `trun.samples[i].sample_duration`                | varint list         |
|  4 | `moofDefaultSampleDuration`        | `tfhd.default_sample_duration`                   | scalar              |
|  5 | `moofSampleCompositionTimeOffsets` | `trun.samples[i].sample_composition_time_offset` | signed varint list  |
|  6 | `moofDefaultSampleSize`            | `tfhd.default_sample_size`                       | scalar              |
|  7 | `moofSampleFlags`                  | `trun.samples[i].sample_flags`                   | varint list (packed; see {{sample-flags}}) |
|  8 | `moofDefaultSampleFlags`           | `tfhd.default_sample_flags`                      | scalar (packed; see {{sample-flags}}) |
|  9 | `moofInitializationVector`         | `senc.samples[i].InitializationVector`           | raw bytes (odd, length-prefixed) |
| 10 | `moofBaseMediaDecodeTime`          | `tfdt.base_media_decode_time`                    | scalar              |
| 11 | `moofSubsampleCount`               | `senc.samples[i].subsample_count`                | varint list         |
| 12 | `moofFirstSampleFlags`             | `trun.first_sample_flags`                        | scalar (packed; see {{sample-flags}}) |
| 13 | `moofBytesOfClearData`             | `senc.samples[i].subsamples[j].BytesOfClearData` | varint list         |
| 14 | `moofSampleCount`                  | `trun.sample_count`                              | scalar              |
| 15 | `moofBytesOfProtectedData`         | `senc.samples[i].subsamples[j].BytesOfProtectedData` | varint list     |
| 16 | `moofPerSampleIVSize`              | `senc.per_sample_iv_size`                        | scalar              |

The ID space is structurally aligned with the parity rule: every
default/scalar field has an even ID and every per-sample list field
has an odd ID, with `moofInitializationVector` (9) as the documented
exception (raw bytes rather than a varint list).

## prft fields {#prft-fields}

Fields whose source is a `ProducerReferenceTimeBox` ({{CMAF}}
§6.6.8) attached to the same CMAF chunk:

| ID | Symbol             | Source field                  | Kind                  |
|----|--------------------|-------------------------------|-----------------------|
| 18 | `prftNtpTimestamp` | `prft.ntp_timestamp` (NTP64)  | scalar (absolute in full; zigzag delta in delta) |
| 20 | `prftMediaTime`    | `prft.media_time`             | scalar (absolute in full; zigzag delta in delta) |
| 22 | `prftVersion`      | `prft.version`                | scalar (default 1)    |
| 24 | `prftFlags`        | `prft.flags` (24-bit FullBox flags) | scalar (default 0) |

A LOCMAF chunk carries `prft` iff at least one of IDs 18, 20, 22,
24 is present in its property block. In a delta chunk, the scalar
fields are zigzag-encoded deltas against the most recent full
chunk in the same group that itself carried `prft` fields. If the
previous full chunk had no `prft`, an encoder that begins emitting
`prft` mid-group MUST use absolute encodings (i.e. re-anchor).

### `prftNtpTimestamp`

The value is the full 64-bit NTP timestamp defined by ISO BMFF
(32-bit seconds since 1900-01-01 + 32-bit fraction) carried as a
varint scalar — absolute in a full chunk, zigzag delta in a delta
chunk. Full source precision is preserved so that downstream
consumers can measure producer-vs-receiver clock drift from the
sub-millisecond jitter around the mean inter-chunk period (a
coarser representation would round away the drift signal).

NTP64 is layout-equivalent to a Q32.32 fixed-point seconds value
(integer seconds in the upper 32 bits, binary fraction of a
second in the lower 32 bits). Encoders and receivers MUST treat
the field as a single 64-bit unsigned integer for the purposes of
delta computation:

~~~
encoder: delta_i64 = (int64)(current_ntp64 - previous_ntp64)
         wire      = MOQT-varint(zigzag(delta_i64))

receiver: delta_i64 = unzigzag(MOQT-varint-decode(wire))
          current_ntp64 = previous_ntp64 + (uint64)delta_i64
~~~

The carry from fraction into seconds at a second boundary is
absorbed by the 64-bit add naturally; there is no separate
handling. The receiver splits the resulting 64-bit value back
into the `prft.ntp_timestamp` seconds (upper 32) and fraction
(lower 32) fields.

The steady-state delta for common frame periods lands in the
4-byte varint band: ~85.9 M units for a 20 ms (50 fps) gap, ~71.6 M
for 60 fps, ~171.8 M for 25 fps. The 4-byte cost is the dominant
per-chunk overhead of the prft path; encoders that do not need
drift-detection precision MAY choose the per-group emission
pattern instead of per-chunk (see {{prft}}).

### `prftMediaTime`

`prftMediaTime` carries the v1 `prft.media_time` field (an integer
in the track's `mdhd.timescale` ticks) directly. It is not
re-scaled. Steady-state deltas at the track timescale are small
(e.g. 1024 ticks for an AAC frame at 48 kHz, 1001 ticks for a
60000/1001 fps video frame at timescale 60000) and fit in 1–2
varint bytes.

### `prftVersion`

Defaults to 1 (the v1 `prft` form with a 64-bit `media_time`).
Encoders SHOULD omit the field; receivers that find the field
absent reconstruct version 1.

### `prftFlags`

Carries the 24-bit FullBox `flags` field defined in
ISO/IEC 14496-12 for `ProducerReferenceTimeBox`. Known values
include 0 (wall-clock anchor at the encoder, the common case), 1,
2, 4, 8, and 24 (combinations of the lower bits identifying
inband-event semantics and producer scope). Encoders MAY omit the
field when its value is 0; receivers that find the field absent
reconstruct flags = 0. The on-wire encoding is a varint scalar;
LOCMAF does not constrain the value beyond what {{ISOBMFF}}
defines.

### `prft.reference_track_ID`

Not carried on the wire. The receiver reconstructs it as the
`track_ID` of the single `trak` in the CMAF Header's `moov` (see
{{scope}}).

## styp fields {#styp-fields}

Fields whose source is the `FileTypeBox` payload of a `styp` box
({{ISOBMFF}}, §4.3) at the top of a CMAF chunk:

| ID | Symbol             | Kind                              | Notes                                                                                  |
|----|--------------------|-----------------------------------|----------------------------------------------------------------------------------------|
| 23 | `stypBrandList`    | raw bytes (odd, length-prefixed) | Concatenation of 4-byte FourCC codes: `major_brand` followed by each `compatible_brand`. Length MUST be a positive multiple of 4. |

LOCMAF does not carry `styp.minor_version` on the wire. Per
{{CMAF}} §7.2, `minor_version` is 0 for any structural CMAF brand
used as `major_brand`, so the reconstructed `styp.minor_version`
is always 0.

`styp` fields MAY appear only in `LocmafFullHeader`; encoders MUST
NOT emit them in `LocmafDeltaHeader`. Per {{CMAF}} §7.3.3.1, a
`styp` inside an addressable media object is ignored by players, so
the delta path has no use for it.

If a `LocmafFullHeader` carries no `stypBrandList`, the receiver
emits no `styp` box in the reconstructed CMAF chunk. CMAF
({{CMAF}} §7.3.3.1) does not require a `styp` for decoding or
playback; players that need brand information consult the CMAF
Header's `ftyp`. Per {{CMAF}} §7.2, when an encoder *does* emit
`stypBrandList`, the reconstructed `styp.minor_version` is 0.

## emsg field {#emsg-fields}

| ID | Symbol     | Kind                              | Notes                                                                                              |
|----|------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| 25 | `emsgList` | raw bytes (odd, length-prefixed) | A self-delimited concatenation of v1 `emsg` records in CMAF order. See {{emsg}} for record format. |

The presence of `emsgList` is independent of `LocmafFullHeader`
vs `LocmafDeltaHeader`. Both kinds carry the full list when
present; there is no delta encoding for `emsg`.

## Delta deletion marker {#deletion-marker}

| ID | Symbol                       | Kind                              | Notes                                            |
|----|------------------------------|-----------------------------------|--------------------------------------------------|
| 27 | `moofDeltaDeletedLocmafIDs`  | varint list (odd, length-prefixed) | List of field IDs removed since the previous moof in the same group. Used only in `LocmafDeltaHeader`. |

# Full LOCMAF Chunk Encoding {#full-chunk}

A `LocmafFullHeader` carries an absolute encoding of one CMAF
chunk's head: at most one optional `styp`, at most one optional
`prft`, zero or more `emsg` boxes that preceded the `moof` in the
source, and the `moof` itself. The `mdat` payload follows the
property block, unchanged.

## Emission rules for moof fields

The encoder walks the source `moof` (paired with the catalog's
`moov`) and emits each moof field only when the value cannot be
derived from the `moov`'s `trex` defaults:

| Field                              | Emitted when                                                                              |
|------------------------------------|-------------------------------------------------------------------------------------------|
| `moofSampleDescriptionIndex`       | `tfhd.HasSampleDescriptionIndex()` AND value ≠ `trex.default_sample_description_index`    |
| `moofDefaultSampleDuration`        | `tfhd.HasDefaultSampleDuration()` AND value ≠ `trex.default_sample_duration`              |
| `moofDefaultSampleSize`            | `tfhd.HasDefaultSampleSize()` AND value ≠ `trex.default_sample_size` AND `sample_count > 1` |
| `moofDefaultSampleFlags`           | `tfhd.HasDefaultSampleFlags()` AND value ≠ `trex.default_sample_flags`                    |
| `moofBaseMediaDecodeTime`          | always                                                                                    |
| `moofSampleCount`                  | always                                                                                    |
| `moofFirstSampleFlags`             | `trun.HasFirstSampleFlags()`                                                              |
| `moofSampleSizes`                  | `trun.HasSampleSize()` AND `sample_count > 1`                                             |
| `moofSampleDurations`              | `trun.HasSampleDuration()`                                                                |
| `moofSampleCompositionTimeOffsets` | `trun.HasSampleCompositionTimeOffset()`                                                   |
| `moofSampleFlags`                  | `trun.HasSampleFlags()`                                                                   |
| `moofPerSampleIVSize`              | `senc` present AND `per_sample_iv_size ≠ tenc.default_per_sample_iv_size`                 |
| `moofInitializationVector`         | `senc` present AND `per_sample_iv_size > 0` AND samples carry IVs (see also {{cenc-iv}})  |
| `moofSubsampleCount`               | `senc` present AND samples carry subsample maps                                           |
| `moofBytesOfClearData`             | same as `moofSubsampleCount`                                                              |
| `moofBytesOfProtectedData`         | same as `moofSubsampleCount`                                                              |

In strict `cmf2` mode (see {{scope}}), `moofDefaultSampleDuration`,
`moofDefaultSampleSize`, `moofDefaultSampleFlags`, and
`moofSampleDescriptionIndex` are emitted unconditionally on the
full chunk, even when they match `trex`.

When `sample_count == 1`, both `moofSampleSizes` (ID 6) and
`moofDefaultSampleSize` (ID 2) MUST be omitted; the single
sample's size is derivable from the MOQT object's mdat-payload
length (object length minus the framing already consumed), so
neither field carries new information. The receiver MUST use this
derivation and MUST NOT consult `trex.default_sample_size` for
single-sample chunks.

## Emission rules for prft / styp / emsg fields

`stypBrandList` is emitted iff the source CMAF chunk preceded its
`moof` with a `styp` box. When omitted, the receiver produces no
`styp` in the reconstructed chunk.

`prft` fields (18, 20, 22, 24) are emitted iff the source CMAF
chunk preceded its `moof` with a `prft` box, subject to the
per-field defaults described in {{prft-fields}} (encoders MAY omit
`prftVersion` when it is 1 and `prftFlags` when it is 0). All
emitted values are absolute.

`emsgList` is emitted iff the source CMAF chunk preceded its `moof`
with one or more v1 `emsg` boxes. See {{emsg}} for the record
format.

# Delta LOCMAF Chunk Encoding {#delta-chunk}

A `LocmafDeltaHeader` carries only the differences between the
current CMAF chunk's head and the most recently received full
chunk in the same MOQT group.

## Field value encoding

Each emitted field's value is interpreted relative to its kind:

| Kind                  | Wire encoding                                                                                  | Reconstruction                                |
|-----------------------|------------------------------------------------------------------------------------------------|-----------------------------------------------|
| scalar (even ID)      | zigzag varint (see {{zigzag}}) of `current_value − previous_value`                             | `current = previous + delta`                  |
| varint list (odd ID)  | zigzag varint (see {{zigzag}}) per element, concatenated; element delta = `current[i] − previous[i]` | element-wise sum with the previous list  |
| raw bytes (odd ID)    | full new bytes verbatim                                                                        | overwrite previous bytes                      |

The "previous value" for each field is the effective value used in
the reconstruction of the previous LOCMAF chunk in the same group
(or, after a mid-group `LocmafFullHeader`, the previous chunk
starting from that re-anchor).

### List length changes {#list-length-changes}

The length of a per-sample list field (IDs 1, 3, 5, 7, 11) in the
current chunk is `moofSampleCount`, which is always emitted (see
{{full-chunk}}). The per-subsample list fields (IDs 13, 15) have
total length equal to `sum(moofSubsampleCount[i])` over the new
sample count. Consequently the receiver knows
`len(current)` for every list field before parsing the field's
payload bytes.

When `len(current) ≠ len(previous)` the delta rule extends as
follows:

- For indices `i` in `[0, min(len(current), len(previous)))`: the
  wire carries `zigzag(current[i] − previous[i])` and the receiver
  reconstructs `current[i] = previous[i] + delta[i]`.
- For indices `i` in `[len(previous), len(current))` (current
  longer than previous): the wire carries `zigzag(current[i])`,
  i.e. the absolute value, equivalent to treating the missing
  previous entry as 0. The receiver reconstructs
  `current[i] = delta[i]`.
- For indices `i` in `[len(current), len(previous))` (current
  shorter than previous): no bytes are emitted for these
  positions. The receiver simply truncates to `len(current)`.

The deletion list `moofDeltaDeletedLocmafIDs` (ID 27) is an
exception: it carries the set of field IDs deleted in *this*
chunk, encoded as plain unsigned varints (not zigzag, not deltas
against a "previous deletion list"). Its length is determined by
the field's byte-length prefix; the receiver reads unsigned
varints until the prefix is exhausted.

## `moofBaseMediaDecodeTime` is normally derived

The receiver derives the new BMDT as
`previous_bmdt + sum(previous_sample_durations)`. When the source
BMDT diverges from this derivation (audio pre-roll, splicing,
stream re-anchor), the encoder MUST emit `moofBaseMediaDecodeTime`
(ID 10) in the delta chunk as an absolute unsigned varint (i.e.
the same encoding as in a full chunk, not a zigzag delta). The
receiver checks for the field first and uses its value when
present.

## Deletions

Delta encoding in LOCMAF is additive: a field absent from a
`LocmafDeltaHeader` is treated as unchanged from the previous
chunk. This compresses the common case (most moof fields stay
stable from chunk to chunk) but on its own gives an encoder no way
to signal that a field which was present in the previous chunk is
genuinely gone in the current one. The deletion marker provides
that signal.

The motivating case is `moofFirstSampleFlags` (ID 12). A SAP-1
random-access chunk emits this field to flag its first sample as
a sync sample; the immediately following non-sync chunk must say
"this override no longer applies" so the receiver falls back to
`trex.default_sample_flags` for the first sample.

The `moofDeltaDeletedLocmafIDs` field (ID 27) carries a varint
list of field IDs that were present in the previous chunk but are
no longer present. The decoder applies deletions before applying
deltas. Example: when the first chunk of a group carried
`moofFirstSampleFlags` (a SAP-1 sync sample) and the second chunk
does not, the second chunk emits ID 27 with value `[12]`. The
typical cost is two bytes — one length-prefixed list with one
field ID — versus the tens to hundreds of bytes of re-anchoring.

## Empty delta

An empty delta payload (`properties_length == 0`) is valid and
means "no field changed since the previous chunk." This is the
steady-state case for sample-level fragmented streams. The
on-wire LOCMAF object reduces to `LocmafDeltaHeader |
properties_length=0 | mdat`, which is two bytes plus the mdat.

## `prft` and `emsg` in delta chunks

`prft` fields use the encoding above (scalar values become zigzag
deltas against the in-group reference). `emsgList` carries the
full new event list, with no delta encoding — see {{emsg}}.

`stypBrandList` (ID 23) MUST NOT appear in a `LocmafDeltaHeader`.

# Compact `sample_flags` Encoding {#sample-flags}

ISO BMFF `sample_flags` ({{ISOBMFF}} §8.8.3.1) is a 32-bit bit-
packed field, but the bits that vary in CMAF content occupy only
five of the 32. LOCMAF encodes the five varying bits in a 5-bit
transport value to fit them in a single 6-bit-payload MOQT varint
(leaving room for the zigzag sign bit in delta context).

## Wire encoding

The 5-bit packed value (LSB first):

| bit  | source field                |
|------|-----------------------------|
| 0    | `sample_is_non_sync_sample` |
| 1–2  | `sample_depends_on`         |
| 3–4  | `sample_is_depended_on`     |

`moofSampleFlags` (ID 7) carries this 5-bit value (range 0–31) per
sample. `moofDefaultSampleFlags` (ID 8) and `moofFirstSampleFlags`
(ID 12) carry the same 5-bit transport.

In a full chunk the field is an unsigned varint scalar (or list).
In a delta chunk it is a signed zigzag varint (or list of zigzag
deltas).

## Reconstruction

The receiver expands the 5-bit transport into a 32-bit
`sample_flags`:

~~~
sample_flags = (is_depended_on << 22)
             | (depends_on     << 24)
             | (non_sync       << 16)
~~~

`is_leading`, `sample_has_redundancy`, `sample_padding_value`, and
`sample_degradation_priority` are reconstructed as zero.

## Encoder constraint

The encoder MUST validate that the source's `sample_flags`
populates only the five bits listed above (see {{scope}}). Source
content that uses other bits MUST be carried via plain CMAF
packaging instead.

# `prft` Round-Trip {#prft}

`prft` ({{CMAF}} §6.6.8, §7.3.2.4) carries an NTP-style wall-clock
anchor tied to a media time. In CMAF it MAY precede any `moof`
inside a CMAF chunk; the `prft` applies to the addressable media
object whose `moof` it precedes. LOCMAF carries it per-chunk via
field IDs 18, 20, 22, and 24 inside the chunk header (see
{{prft-fields}}).

The reconstructed `prft.reference_track_ID` is always the
`track_ID` of the single `trak` in the CMAF Header `moov`; no
field is needed on the wire.

## Presence

`prft` is optional per chunk. The receiver reconstructs a `prft`
box in the output CMAF chunk iff at least one `prft` field (18, 20,
22, or 24) is present in the chunk header. Three producer patterns
are
supported by this presence-signalling without further wire-format
support:

1. **None:** no `prft` field ever emitted.
2. **Per-group:** absolute `prft` fields emitted on the
   `LocmafFullHeader` only; absent from subsequent
   `LocmafDeltaHeader` objects in the group.
3. **Per-chunk:** absolute `prft` fields on the
   `LocmafFullHeader`, delta `prft` fields on subsequent
   `LocmafDeltaHeader` objects.

## Delta semantics

In a delta chunk, `prftNtpTimestamp` (ID 18) and `prftMediaTime`
(ID 20) are zigzag-signed varint deltas against the most recent
full chunk in the same group that carried the same field. Deltas
are signed because both quantities can decrease in valid CMAF
streams: producer NTP clocks can be corrected backward, and
composition-time reordering with B-frames can place a chunk's
presentation anchor before the previous chunk's.

# `emsg` Round-Trip {#emsg}

The DASH event message box ({{DASH}} §5.10.3) carries application-
defined timed metadata. CMAF (§7.4.5) mandates version 1 emsg
boxes for in-band CMAF event messages.

LOCMAF carries `emsg` as a length-prefixed list of records inside
the chunk header at field ID 25. Encoders MUST emit only v1
records.

## Relationship to MSF `eventtimeline`

New MOQT deployments SHOULD use a companion MSF `eventtimeline`
track {{MSF}} for event metadata rather than inline `emsg`. LOCMAF's
`emsg` support exists primarily to preserve inline events in
sources transcoded from DASH or CMAF Ingest {{DASH-IF-INGEST}}.

## Record format

Each record inside `emsgList` is a compact encoding of a v1 `emsg`
payload:

~~~
record = scheme_id_uri          (varint length + UTF-8 bytes)
       | value                  (varint length + UTF-8 bytes)
       | timescale              (varint; 0 = "use track mdhd.timescale")
       | presentation_time      (encoding depends on timescale, see below)
       | event_duration         (varint, in `timescale` ticks; 0 = unknown)
       | id                     (varint)
       | message_data           (varint length + opaque bytes)
~~~

`reference_track_id` and `version` are implicit (the track this
chunk belongs to, and 1 respectively).

### `timescale` default

`timescale == 0` means "use the track's `mdhd.timescale`." Receivers
write the track's `mdhd.timescale` into the reconstructed `emsg.timescale`
field when the record carried 0. Encoders MAY emit a non-zero
`timescale` only when the source's `emsg.timescale` actually
differs from the track timescale.

### `presentation_time` encoding

The encoding of `presentation_time` depends on the record's
`timescale` field:

- **`timescale == 0` (track-timescale, delta encoding):** the field
  is encoded as a signed zigzag varint delta against the chunk's
  BMDT (in track-timescale ticks). The reconstructed value is
  `chunk_bmdt + delta`.
- **`timescale != 0` (foreign-timescale, absolute encoding):** the
  field is encoded as an unsigned varint carrying the absolute
  `emsg.presentation_time` directly. No delta is applied because
  the BMDT and the event time would live on different axes.

The receiver discriminates by inspecting the record's `timescale`
field, which is encoded earlier in the record.

# DRM Box Round-Trip {#drm}

LOCMAF preserves the per-sample CENC {{CENC}} metadata needed for
EME-based decryption.

## Supported schemes

LOCMAF v0.2 supports the following CENC {{CENC}} protection schemes,
identified by the `tenc.default_isProtected = 1` track defaults and
the four-character `scheme_type` in the surrounding `schm` box:

- `cenc`: AES-128-CTR full-sample encryption. Per-sample
  initialization vectors are big-endian counters advanced by the
  per-sample encrypted-byte total ({{CENC}}, §10.1); LOCMAF carries
  these IVs via `moofInitializationVector` and permits omission
  under the counter rule of {{cenc-iv}}.
- `cbcs`: AES-128-CBC subsample pattern encryption with a constant
  initialization vector taken from `tenc.default_constant_iv`
  ({{CENC}}, §10.4); no per-sample IV appears in `senc`, and the
  pattern (`default_crypt_byte_block` / `default_skip_byte_block`)
  is carried verbatim with the CMAF Header.

The `cbc1` and `cens` schemes are out of scope; sources using them
MUST fall back to plain CMAF packaging.

## Supported boxes

| Box   | Where in CMAF          | LOCMAF treatment                                                                       |
|-------|------------------------|----------------------------------------------------------------------------------------|
| `senc`| inside `traf`          | per-sample IVs and subsample maps carried via moof field IDs 9, 11, 13, 15, 16.        |
| `saio`| inside `traf`          | not carried on the wire; recomputed by the receiver to point at the reconstructed `senc`. |
| `saiz`| inside `traf`          | not carried on the wire; reconstructed from per-sample IV size and subsample counts.   |
| `tenc`| inside `sinf` in `moov`| carried verbatim inside the CMAF Header.                                               |

## Unsupported boxes

The following CMAF DRM boxes are not supported by LOCMAF v0.2.
Sources that require them MUST use plain CMAF packaging instead:

| Box   | Reason for exclusion                                                                                                      |
|-------|---------------------------------------------------------------------------------------------------------------------------|
| `sgpd`/`sbgp` | Mid-fragment key rotation via `seig` sample groups is out of scope; KID changes MUST align with fragment boundaries (see {{scope}}). |
| `pssh` (per-fragment) | License-acquisition information is signalled via the CMSF `contentProtections` mechanism, per {{CMAF}} §7.4.3. |
| `subs`        | Sub-sample information for image subtitle profiles (e.g. `im1i`) is out of scope.                                 |

## CENC IV counter prediction (optional) {#cenc-iv}

For the `cenc` scheme, ISO/IEC 23001-7 §9.6 specifies
that the per-sample initialization vector is a big-endian counter
advanced sample-by-sample by exactly
`ceil(total_encrypted_bytes_in_sample / 16)` AES blocks. Both
endpoints already see the per-sample encrypted-byte totals
(`moofBytesOfProtectedData`) and the IV anchor (carried on the
first full chunk of the track), so the receiver can derive every
subsequent per-sample IV deterministically.

LOCMAF v0.2 permits encoders to omit `moofInitializationVector` (ID
9) when the source follows this counter rule, and requires
receivers to support derivation:

- An encoder MAY omit `moofInitializationVector` on full and delta
  chunks when every per-sample IV in the chunk matches the value
  derived by the CENC counter rule from the previous chunk's IVs
  and `moofBytesOfProtectedData` totals.
- A receiver MUST be able to derive per-sample IVs from the counter
  rule. When `moofInitializationVector` is absent and the scheme is
  `cenc`, the receiver advances the running IV counter and uses the
  derived value.
- When the source diverges from the counter rule (random IVs,
  mid-track counter restart, or any non-conformant strategy), the
  encoder MUST emit `moofInitializationVector` absolutely on every
  affected sample.

The `cbcs` scheme uses a constant IV from `tenc.default_constant_iv`
carried once via the CMAF Header. There is no per-sample IV in the
moof in the first place, so counter prediction does not apply to
`cbcs`.

# Event-Only Tracks and CMAF Ingest Compatibility {#event-only}

DASH-IF Ingest {{DASH-IF-INGEST}} defines a CMAF-based push
interface for live encoders. One of its track shapes is the sparse
event-only track: a CMAF track that carries no media samples (or
samples of zero size) and exists purely to deliver timed events
via `emsg` boxes attached to its chunks.

LOCMAF supports event-only tracks without any wire-format
extension. A `LocmafFullHeader` for an event-only group sets
`moofSampleCount = 0`, carries `moofBaseMediaDecodeTime` and
`emsgList`, and is followed by an empty `mdat` payload. Subsequent
chunks in the same group use `LocmafDeltaHeader` with the
absolute-BMDT override pattern (see {{delta-chunk}}) because the
zero sample-count produces no derivation increment.

Two encoder strategies are valid:

1. **Absolute BMDT per chunk.** The delta chunk emits
   `moofBaseMediaDecodeTime` explicitly. Costs an extra varint per
   chunk; recommended for sparse event-only tracks.
2. **Synthetic per-chunk sample.** The encoder sets
   `sample_count = 1` with a `default_sample_duration` equal to
   the intended per-chunk advancement and a zero-size sample. BMDT
   derivation works without override. Matches how DASH-IF Ingest
   commonly shapes sparse metadata tracks (`urim`, `stpp`).

For new MOQT deployments, MSF `eventtimeline` {{MSF}} is the
preferred mechanism for event metadata. LOCMAF event-only tracks
are intended for gateways that transit-relay CMAF Ingest content
unchanged across MOQT.

# Receiver Reconstruction

A receiver maintains, per subscribed track:

1. The track's CMAF Header (from the catalog), parsed for `trex`
   defaults, `tenc` defaults, and `mdhd.timescale`.
2. An in-group "previous chunk" state, populated from the most
   recent `LocmafFullHeader` and updated by each subsequent
   `LocmafDeltaHeader`. Discarded on group boundaries and on
   mid-group `LocmafFullHeader` re-anchors (see {{dispatch}}).

For each LOCMAF object, the receiver:

1. Reads `header_id` and dispatches per {{dispatch}}.
2. Reads `properties_length` and the property block.
3. Decodes property tuples per the parity rule and the per-field
   rules above.
4. Applies the decoded fields to the previous-chunk state to
   produce the absolute field values for the current chunk.
5. Reconstructs the CMAF chunk:
   1. Synthesises a `styp` box from `stypBrandList` when present;
      omits it otherwise.
   2. Synthesises a `prft` box from any `prft` fields present;
      omits it when no `prft` field is present.
   3. Synthesises one or more v1 `emsg` boxes from `emsgList` when
      present; omits them otherwise.
   4. Synthesises the `moof` box (`mfhd`, `traf` with `tfhd`,
      `tfdt`, `trun`, optionally `senc`/`saio`/`saiz`) from the
      moof fields and the CMAF Header's `trex` / `tenc` defaults.
   5. Wraps the mdat payload bytes in an 8-byte `mdat` box header.
6. Feeds the reconstructed chunk to the local CMAF reader / MSE
   pipeline.

The reconstructed CMAF chunk is **functionally equivalent** to the
source chunk: every sample has the same size, decode time,
presentation time, flags, and CENC metadata, and the chunk feeds an
MSE / EME pipeline identically to the source. Byte-level identity
with the source `moof` is not preserved. Implementations MAY differ
in:

- The exact ordering of `saio` / `saiz` / `senc` and other generated
  boxes inside the reconstructed `traf`, provided the ordering is
  legal CMAF.
- The `trun.tr_flags` packing chosen on reconstruction.
- Whether `tfhd` defaults that match `trex` appear in the
  reconstructed `tfhd` (they SHOULD when the encoder ran in strict
  `cmf2` mode; they MAY otherwise).
- The reconstructed `mdat` box header bytes (which the receiver
  synthesises rather than carrying on the wire).

A receiver MUST NOT depend on byte-level identity with the source
CMAF stream. A downstream consumer that needs a specific CMAF
byte layout MUST repackage the output of the LOCMAF receiver to
produce the desired form.

# Security Considerations

LOCMAF is a compression layer over CMAF media and does not
introduce new authentication or confidentiality mechanisms. It is
intended to be used over MOQT {{MOQT}}, which inherits QUIC's
transport security. Per-sample encryption metadata defined by
{{CENC}} is preserved through the LOCMAF round-trip; LOCMAF
neither weakens nor strengthens the underlying DRM scheme.

A receiver MUST validate that reconstructed `moof`, `prft`, and
`emsg` boxes are well-formed before passing them to a media
pipeline. Malformed deltas could otherwise be used to construct
ISO BMFF {{ISOBMFF}} boxes with inconsistent field lengths.
Specifically:

- The receiver MUST bound the size of any reconstructed per-sample
  list against `moofSampleCount`.
- The receiver MUST verify that the sum of reconstructed
  `moofBytesOfClearData` and `moofBytesOfProtectedData` for each
  sample equals the sample's size.
- The receiver MUST verify that the CENC-IV counter, if derived,
  does not advance past the per-sample-IV-size range.

The CENC IV counter-prediction optimisation ({{cenc-iv}}) does not
disclose key material and produces the same IV stream a conformant
encoder would have transmitted; it does not weaken CENC.

Replay considerations within a MOQT group are inherited from MOQT —
LOCMAF adds no new replay attack surface.

# IANA Considerations {#iana}

This document requests IANA to register the following values.

## CMSF Catalog Properties

\[TODO: Register `locmafVersion` in the CMSF catalog property
registry once that registry is defined.]

| Property        | Value type | Defined in     |
|-----------------|------------|----------------|
| `locmafVersion` | string     | this document  |

The value `"0.2"` identifies the wire format specified by this
document.

## LOCMAF Top-Level Header IDs

This document defines a new registry for LOCMAF top-level header
IDs.

| ID    | Symbol               | Reference     |
|-------|----------------------|---------------|
| 23    | `LocmafFullHeader`   | this document |
| 25    | `LocmafDeltaHeader`  | this document |

All other IDs in the unsigned varint range are available for
assignment via Specification Required ({{?RFC8126}}).

LOCMAF property field IDs are part of this document's wire format
and are not registered with IANA. New field IDs are introduced
through revisions of this specification, signalled by a bump of
the `locmafVersion` catalog value (see {{catalog}}). The full
field-ID assignment for this version is given in {{field-ref}}.


--- back

# Acknowledgments
{:numbered="false"}

LOCMAF was developed as part of the Master Thesis work of Hugo
Björs at Eyevinn Technology, supervised by Torbjörn Einarsson.
The authors thank the Media over QUIC working group, in particular
the authors and contributors to {{MOQT}} and {{CMSF}}, for the prior
art this work builds on.
