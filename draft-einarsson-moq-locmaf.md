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

normative:
  MOQT: I-D.draft-ietf-moq-transport-18
  CMSF: I-D.draft-ietf-moq-cmsf-01
  MSF:  I-D.draft-ietf-moq-msf-01
  LOC:  I-D.draft-ietf-moq-loc-02
  CMAF:
    title: "Information technology — Multimedia application format (MPEG-A) — Part 19: Common media application format (CMAF) for segmented media"
    seriesinfo:
      ISO/IEC: 23000-19:2024
    date: 2024
  ISOBMFF:
    title: "Information technology — Coding of audio-visual objects — Part 12: ISO base media file format"
    seriesinfo:
      ISO/IEC: 14496-12:2024
    date: 2024
  CENC:
    title: "Information technology — MPEG systems technologies — Part 7: Common encryption in ISO base media file format files"
    seriesinfo:
      ISO/IEC: 23001-7:2024
    date: 2024

informative:
  DASHIF-ECCP:
    target: https://dashif.org/docs/DASH-IF-ECCP-v1.0.0.pdf
    title: "DASH-IF Implementation Guidelines: Encryption and Content Protection (ECCP)"
    date: 2023
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
QUIC), a compact packaging for low-latency CMAF media carried
end-to-end as MoQ Transport (MOQT) Object payloads, with per-object
overhead comparable to the Low Overhead Container (LOC). LOCMAF
carries the CMAF chunk head metadata from a single `moof` (movie
fragment) as a small set of tagged fields, while leaving the sample
data (`mdat`) untouched. Boxes that may surround the `moof` in a
CMAF chunk — `styp` (segment type), `prft` (producer reference
time), and any number of `emsg` (event message) boxes — are carried
verbatim, each through a generic box element (a genBox). The first Object of
each MOQT group carries a full reference; subsequent Objects in the
same group carry only the differences. The receiver reconstructs
CMAF chunks that are decode-equivalent to the sender input,
including the encryption metadata required by CMAF DRM (Common
Encryption) pipelines, and a canonical byte-identical
reconstruction — independent of the encoder's representation
choices — is defined for conformance testing.

--- middle

# Introduction

CMAF {{CMAF}} chunk headers have a size starting at around 100 bytes,
while the codec frames they describe may be only a few hundred bytes
at low latency and low bitrate, such as audio tracks. Carrying CMAF
directly over MoQ Transport {{MOQT}} therefore incurs a per-object
overhead that the Low Overhead Container (LOC) {{LOC}} avoids by
carrying raw codec frames with a minimal set of metadata. LOC,
however, cannot transport the per-sample CENC {{CENC}} metadata
needed for browser EME / CDM decryption of DRM-protected live
streams, nor the `prft` (Producer Reference Time) and `emsg` (DASH
Event Message) boxes that a CMAF chunk may carry alongside the
`moof`.

LOCMAF closes this gap. It is a packaging — a compact container for
CMAF media — that exploits the observation that consecutive CMAF
chunk heads within a single CMAF segment are nearly identical: the
first chunk of a MOQT group is sent in full, subsequent chunks are
sent as compact deltas against the previous chunk in the same group,
and `mdat` payloads are passed through unchanged. The receiver
reconstructs full CMAF chunks suitable for any unmodified CMAF
playback pipeline (such as browser MSE / EME), or feeds the
elementary samples directly to a frame-based decoder interface
(such as WebCodecs); see {{consumption}}.

LOCMAF is carried end-to-end as the payload of MOQT Objects. MOQT
relays forward the Object payload unchanged; only the encoder
produces LOCMAF and only the receiver expands it. LOCMAF is
therefore a media packaging, not a hop-by-hop transfer encoding, and
not a transport — MOQT {{MOQT}} is the transport.

This document specifies the LOCMAF Object encoding, the generic box
and raw-boxes elements, the full and delta chunk encodings, the
catalog signalling (deferred to {{CMSF}}), the canonical CMAF
reconstruction, and the DRM box round-trip. A reference
implementation is available {{MOQLIVEMOCK}}. Worked examples and
diagrams are published at {{LOCMAF-SITE}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Throughout this document, `vi64` denotes a variable-length integer
as defined in {{Section 1.4.1 of MOQT}}.

The following terms are used throughout this document:

CMAF chunk:
: One `moof` + one `mdat` pair, optionally preceded by at most one
  `styp`, at most one `prft`, and zero or more `emsg` boxes, as
  defined in {{CMAF}} §7.3.3.2. The smallest CMAF addressable media
  object.

CMAF fragment:
: One or more CMAF chunks whose first chunk starts at a Stream Access
  Point ({{CMAF}} §7.3.2.2). A fragment is logically a single
  `MovieFragmentBox` worth of samples; in "chunked" CMAF the samples
  are split across multiple smaller `moof` + `mdat` pairs.

CMAF segment:
: One or more CMAF fragments in decode order ({{CMAF}} §7.3.2.4). The
  segment is the typical unit of HTTP delivery in DASH and HLS-fMP4;
  in LOCMAF the segment corresponds to one MOQT group.

CMAF Header:
: The `ftyp` + `moov` pair that initialises a CMAF track. Also called
  an *initialisation segment* in DASH parlance and carried as
  initialisation data in CMSF {{CMSF}} catalogs.

MOQT group, MOQT object:
: As defined in {{MOQT}}.

LOCMAF Object:
: A MOQT Object whose payload is a LOCMAF Object encoding: either a
  sequence of generic box elements (see below) and exactly one
  moof-header element, followed by the `mdat` payload, or a single
  raw-boxes element, as defined in {{object-encoding}}.

genBox:
: A generic box element that carries one ISO BMFF box that sits
  outside `moof` in a CMAF chunk (for example `styp`, `prft`, `emsg`,
  or `uuid`). See {{genbox}}.

rawBoxes:
: A raw-boxes element that carries one or more complete ISO BMFF
  boxes verbatim, including their box headers, as the entire payload
  of a LOCMAF Object. See {{rawboxes}}.

locmafHeader:
: The single moof-header element of a LOCMAF Object. It carries the
  tagged `moof` fields in either a full (absolute) or delta encoding
  and marks the position of `moof` in the reconstructed chunk. See
  {{object-encoding}}.

Full LOCMAF chunk:
: A LOCMAF Object whose moof-header element is a full header. It
  carries an absolute encoding of the CMAF chunk head and serves as
  the in-group reference for subsequent delta Objects. See
  {{full-chunk}}.

Delta LOCMAF chunk:
: A LOCMAF Object whose moof-header element is a delta header. It
  encodes differences against the most recently received full LOCMAF
  chunk in the same MOQT group. See {{delta-chunk}}.

BMDT:
: Abbreviation for `tfdt.baseMediaDecodeTime` ({{ISOBMFF}}).

# MOQT Group / Object Mapping {#mapping}

LOCMAF assumes the following mapping from CMAF to MOQT:

- One MOQT group per CMAF segment. Group boundaries align with random
  access points.
- One MOQT Object per CMAF chunk. Each MOQT Object is a LOCMAF Object
  carrying the (full or delta) chunk head followed by the unmodified
  `mdat` payload.
- Audio MOQT groups typically have the same duration as the video
  MOQT groups with which they will be muxed, to enable joint tune-in.
- Sparse tracks, such as subtitle, event, or metadata tracks, are
  more likely to have groups that are not aligned with video.

Delta chunks reference the preceding chunk in the same group (see
{{delta-chunk}}), so LOCMAF depends on in-order delivery within a
group. MOQT {{MOQT}} guarantees ordering only within a *subgroup*:
a subgroup carries Objects of one group in ascending Object ID
order on a single stream, while datagrams are unordered. A
publisher MUST therefore send all Objects of a LOCMAF group in a
single subgroup, and MUST NOT use the Datagram forwarding
preference for LOCMAF tracks. The first moof-carrying Object of
each group MUST be a full LOCMAF chunk so a subscriber tuning in at
a group boundary has a complete reference (see
{{element-sequence}}).

When a receiver detects that Objects are missing within a group — a
gap in Object IDs, or a reset of the subgroup's stream — it MUST
NOT apply subsequent delta chunks; it resumes either at the next
full LOCMAF chunk or rawBoxes Object ({{rawboxes}}) in the same
group or at the start of the next group.

# Catalog Signalling {#catalog}

LOCMAF media is signalled in a CMSF {{CMSF}} catalog. This document
extends the allowed `packaging` values defined in {{MSF}} with one
new entry, in the same manner that {{CMSF}} adds `"cmaf"`:

| Name   | Value    | Reference     |
|--------|----------|---------------|
| LOCMAF | `locmaf` | This document |

Every track entry in a CMSF catalog that carries LOCMAF-encoded
media MUST declare a `packaging` value of `"locmaf"`. As with
`"cmaf"`, the `"locmaf"` packaging is defined for CMSF catalogs
only, not for plain MSF {{MSF}} catalogs.

The LOCMAF Object encoding is versioned independently of the CMSF
catalog `version`. This document adds one track-level catalog
field, following the field-definition conventions of {{MSF}}:

| Field          | Name            | Location | Required    | JSON Type |
|----------------|-----------------|----------|-------------|-----------|
| LOCMAF version | `locmafVersion` | T        | Conditional | String    |

`locmafVersion` identifies the LOCMAF wire-format version of the
track. It MUST be present when `packaging` is `"locmaf"` and MUST
NOT be present otherwise. The version specified by this document
is `"0.3"`. A receiver MUST NOT subscribe to a LOCMAF track whose
`locmafVersion` it does not support; when the catalog offers the
same source under an alternative packaging, it MAY select that
instead.

The unknown-field rule of {{parity}} covers additive evolution
within a version; `locmafVersion` signals behavioural changes that
reinterpret existing wire syntax, which a receiver cannot detect
from the wire bytes alone.

The CMAF Header for a `locmaf` track is carried by the **same**
mechanism {{CMSF}} uses for a `cmaf` track: an `initDataList` entry
of inline base64 type (defined by {{MSF}}), referenced from the
track entry by `initRef` (see {{cmaf-header}}). LOCMAF defines no init
carriage of its own. A consequence is that a `cmaf` track and a
`locmaf` track wrapping the same source MAY refer to the same
`initData` entry; only the per-chunk Object encoding differs.

DRM is signalled exactly as for a `cmaf` track, by the CMSF
{{CMSF}} root-level `contentProtections` array referenced from the
track entry by `contentProtectionRefIDs`, following the DASH-IF
content-protection model {{DASHIF-ECCP}}. LOCMAF does **not** use the
MSF moq-secure-objects mechanism; the encryption metadata travels
inside the reconstructed CMAF boxes (see {{drm}}). The precise
catalog field names are those defined by {{MSF}} and {{CMSF}};
beyond the `packaging` value `"locmaf"` and the `locmafVersion`
field, LOCMAF introduces no catalog fields.

`cmaf` and `locmaf` tracks MAY be mixed in the same catalog under the
same namespace — for example video using `cmaf` packaging while audio
uses `locmaf` — and MAY coexist as alternative encodings of the same
source.

# CMAF Header Delivery {#cmaf-header}

The CMAF Header for a LOCMAF track is byte-identical to the CMAF
Header a plain `cmaf` track of the same source would carry — `ftyp`
followed by `moov` (which contains `mvex` with `trex`, and `pssh`
when the content is protected). It is delivered uncompressed via
the catalog using the same CMSF {{CMSF}} mechanism as for `cmaf`
packaging (an `initDataList` entry of inline base64 type,
referenced by `initRef`). There is no LOCMAF-specific CMAF Header
carrier.

The `moov` in the CMAF Header MUST contain exactly one `trak` box
(see {{scope}}).

A LOCMAF receiver:

1. Resolves the track's `initRef` to its `initDataList` entry and
   takes the base64-encoded CMAF Header bytes.
2. Base64-decodes the CMAF Header bytes.
3. Feeds the bytes to its MSE / decoder pipeline exactly as it would
   for a plain CMAF track.
4. Extracts the parameters required to reconstruct CMAF chunks — the
   single `trak`'s `track_ID`, `mdhd.timescale`, `trex` defaults, and
   any track-encryption information (`tenc` defaults: default KID,
   default per-sample IV size, constant IV, scheme type, and pattern
   parameters) — from the decoded CMAF Header. These values seed the
   per-track reconstruction state (see {{canonical}}).
5. Begins receiving LOCMAF-encoded media Objects on the subscribed
   track and reconstructs each CMAF chunk from the LOCMAF Object
   payload.

Compression of the catalog itself is out of scope for LOCMAF and
handled at the MOQT {{MOQT}} / MSF {{MSF}} layer.

# Scope and Publisher Requirements {#scope}

LOCMAF targets the low-latency CMAF case: short CMAF fragments
composed of small CMAF chunks (often one sample per chunk),
optionally carrying CENC encryption metadata. To keep the
packaging minimal, the following constraints apply.

## Mandatory preconditions

A LOCMAF publisher MUST ensure that:

1. **Single `trak` per `moov`.** The CMAF Header contains exactly
   one `trak` box. Multi-track ISO BMFF files MUST be demuxed
   before LOCMAF encoding.
2. **No key ID (KID) change within a CMAF chunk.** Key-identifier
   transitions MUST align with fragment (and therefore chunk)
   boundaries. This removes the need for `sgpd` / `sbgp` boxes in
   the packaging.

If a source violates either of these, the publisher MUST NOT use
LOCMAF packaging for that track and MUST instead use plain CMAF
packaging. LOCMAF and plain CMAF tracks MAY coexist in the same
catalog under the same namespace (see {{catalog}}).

LOCMAF places no constraint on `sample_flags`: the full 32-bit ISO
BMFF `sample_flags` value ({{ISOBMFF}} §8.8.3.1) round-trips
through the packaging (see {{field-ref}}).

## Recommended source properties

The following are recommendations whose violation costs bytes but
does not break LOCMAF:

1. **Commensurate media timescales.** Choose a timescale so every
   frame has an exact integer duration (e.g. 48 000 for 48 kHz AAC,
   60 000 for 60000/1001 fps video).
2. **Stable `trex` defaults.** Keeping `trex` consistent across the
   stream maximises what can be omitted from each chunk header.

## Optional encoder modes

A LOCMAF encoder MAY operate in **strict `cmf2` mode**, in which it
always emits the four `tfhd` defaults (sample duration, size,
flags, sample-description index) in the full chunk header even when
they match `trex`. This costs a handful of bytes per group but
produces reconstructed CMAF chunks in which each chunk is a
self-decodable single-chunk fragment per {{CMAF}} §7.7.3. It need
not be signalled, since it does not affect compatibility between
encoders and decoders. Strict `cmf2` mode is an opt-in encoder
tweak and is **not** the canonical baseline used for conformance;
the canonical reconstruction defined in {{canonical}} emits a
`tfhd` default only when it differs from `trex`.

# Object Encoding {#object-encoding}

A LOCMAF Object is the payload of a single MOQT {{MOQT}} object. It
carries one CMAF chunk: the `moof` chunk head as a small set of
tagged fields, any boxes that precede the `moof`, and the
unmodified `mdat` sample data. MOQT relays forward the Object
payload unchanged; only the encoder produces LOCMAF and only the
receiver expands it back to CMAF.

## Element sequence and dispatch {#element-sequence}

A LOCMAF Object payload takes one of two shapes. The first — the
moof-carrying shape — is an ordered sequence of *elements* followed
by the raw `mdat` payload:

~~~
[ genBox ]*       zero or more pre-moof boxes, in reconstruction order
locmafHeader      exactly one: a full or delta moof header
mdat raw payload  length = MOQT-object-len minus the elements above
~~~

The second shape is a single raw-boxes element spanning the entire
payload:

~~~
rawBoxes          complete ISO BMFF boxes, verbatim ({{rawboxes}})
~~~

In the moof-carrying shape, the `locmafHeader` marks where the
`moof` sits in the reconstructed chunk: every `genBox` before it
renders before the `moof`, and the `mdat` immediately follows the
`moof`. The common case — no boxes outside the `moof` — is simply
`locmafHeader` followed by `mdat`; `genBox` is purely additive.

Each element begins with an **element_type** `vi64`. Four element
types are defined:

| element_type | Symbol             | Meaning                                 | Delimited by              |
|:------------:|--------------------|-----------------------------------------|---------------------------|
| 1            | `genBox`           | one generic pre-`moof` box ({{genbox}}) | its own `box_size` field  |
| 2            | `locmafFullHeader` | full moof header (absolute encoding)    | its `properties_length`   |
| 3            | `locmafDeltaHeader`| delta moof header (in-group deltas)     | its `properties_length`   |
| 4            | `rawBoxes`         | complete boxes, verbatim ({{rawboxes}}) | the Object end (sole element) |

The `mdat` payload carries no element_type tag — it is whatever
bytes remain after the `locmafHeader`.

A receiver parses elements in a loop, reading element_type `vi64`
values:

1. If the first element_type of the Object is `4`, the Object is a
   single `rawBoxes` element ({{rawboxes}}) and the loop ends;
   element_type `4` anywhere but first is malformed.
2. While the element_type is `1`, it parses one `genBox` (delimited
   by its `box_size`, see {{genbox}}) and continues.
3. When the element_type is `2` or `3`, it parses exactly one
   `locmafHeader`, full or delta respectively (delimited by its
   `properties_length`, see {{full-chunk}} and {{delta-chunk}}).
   The bytes following that header's property block, to the end of
   the MOQT object, are the `mdat` payload.

Exactly one header element MUST appear in a moof-carrying LOCMAF
Object, and it MUST be the last element before the `mdat` payload.
A `genBox` that follows the header is malformed.

The full-vs-delta distinction is signalled exclusively by the
header element_type (`2` for full, `3` for delta), never by the
MOQT object position within a group:

1. The first moof-carrying object of every MOQT group MUST carry a
   `locmafFullHeader`, so a subscriber tuning in at a group
   boundary has a complete reference. The only objects that may
   precede it in the group are rawBoxes Objects ({{rawboxes}}).
2. The encoder MAY emit a `locmafFullHeader` at any object position
   within a group, not only at object index 0. A mid-group full
   chunk re-anchors the in-group reference for subsequent delta
   chunks.
3. After parsing a `locmafFullHeader`, the receiver MUST discard
   its in-group delta state and treat the new full chunk as the
   reference for any following `locmafDeltaHeader` objects in the
   group.
4. A rawBoxes Object likewise resets the in-group delta state; the
   next moof-carrying object in the group MUST carry a
   `locmafFullHeader` (see {{rawboxes}}).
5. The receiver MUST dispatch on the header element_type alone. It
   MUST NOT infer "full" from object index 0 or "delta" from object
   index greater than 0.

An element_type not defined in the table above is not
self-delimiting, so a receiver cannot skip it. A receiver that
reads an unrecognised
leading element_type MUST treat the Object as malformed and reject
it; there is no generic skip for unknown top-level elements. This
hard failure is deliberate: an element one receiver silently
skipped and another understood would make the two reconstruct
different chunks, defeating canonical comparison ({{canonical}}).
Extension happens through new `genBox` `box_name` FourCCs and new
header field IDs (see {{parity}}) — both of which *are*
self-delimiting — not through new element types.

For event-only tracks (see {{event-only}}) the `mdat` payload MAY
be zero bytes; the receiver reconstructs an empty `mdat` box
(8-byte header only).

## Header element layout {#header-layout}

A `locmafHeader` element (full or delta) is, in order:

~~~
element_type        vi64     = 2 (full) or 3 (delta)
properties_length   vi64     byte length of the property block
property block      properties_length bytes of (field_id, value) tuples
~~~

`element_type` and `properties_length` are `vi64` values. The
property block is the flat sequence of
`(field_id, value)` tuples defined in {{parity}} and is exactly
`properties_length` bytes long, so the header element is
self-delimited. When this is the Object's header element, the bytes
following the property block, to the end of the MOQT object, are the
`mdat` payload (see {{element-sequence}}). A delta header with
`properties_length == 0` carries no field changes (see
{{delta-chunk}}).

## Property encoding (parity rule) {#parity}

The property block inside a `locmafHeader` is a flat sequence of
`(field_id, value)` tuples. Field IDs are `vi64` values.
This is the property scheme of {{LOC}} §2.3: the value encoding is
determined by the parity of the field ID. LOCMAF reuses that parity
scheme but governs its own field IDs in this document (see
{{field-ref}}); they are not {{LOC}} properties and are not
IANA-registered.

- **Even ID — scalar.** The value is a single `vi64` with no
  length prefix. In a full header it is an absolute unsigned
  `vi64`; in a delta header it is a zigzag `vi64` (see {{zigzag}})
  of the signed delta against the in-group reference.
- **Odd ID — length-prefixed bytes.** The tuple is `field_id |
  byte_length | bytes`. The interpretation of the bytes is
  per-field. `vi64`-list fields concatenate their elements, each an
  absolute unsigned `vi64` in full context and a per-element
  zigzag `vi64` (see {{zigzag}}) in delta context. Raw-byte fields
  (`sencInitializationVector`, ID 9) carry opaque content verbatim
  in both contexts.

Parity governs *framing*: it tells a receiver how far every tuple
extends, whether or not it recognises the field ID. The
interpretation of the value bytes is per-field ({{field-ref}}).
Three fields deviate from the absolute-in-full / delta-in-delta
value encoding above:

- `trunSampleCompositionTimeOffsets` (ID 5): elements are zigzag
  `vi64` values (see {{zigzag}}) in **both** full and delta context,
  because composition time offsets are signed in `trun` version 1 —
  the common video case, where B-frames make the composition/
  decode-time relation non-monotonic.
- `sencInitializationVector` (ID 9): opaque raw bytes, carried
  verbatim in both contexts (overwrite, never a delta).
- `deltaDeletedLocmafIDs` (ID 27): delta-only control list whose
  elements are plain unsigned `vi64` field IDs, never zigzag (see
  {{deletion-marker}}).

Two fields are additionally restricted to one header kind:
`tfdtBaseMediaDecodeTime` (ID 10) appears only in full headers — a
delta chunk's BMDT is always derived (see {{bmdt-derivation}}) —
and `deltaDeletedLocmafIDs` (ID 27) appears only in delta headers.

A receiver that encounters a field ID not defined in this document
MUST skip its value using the parity rule — one `vi64` for an even
ID, `byte_length` bytes for an odd ID — and MUST otherwise ignore
it. New field IDs can therefore be added backward-compatibly, as
long as a reconstruction that ignores them remains correct; an
extension that cannot be safely ignored requires a new `packaging`
value instead.

The ID space is structurally aligned with the parity rule: every
scalar/default field has an even ID and every list or byte field
has an odd ID. A field ID MUST NOT appear twice in one property
block; a receiver MUST reject a property block that repeats one.
Field IDs MAY appear in any order and receivers MUST tolerate any
ordering; encoders SHOULD emit IDs in ascending order, and the
canonical encoding ({{canonical-encoding}}) requires it.

## Zigzag vi64 encoding {#zigzag}

A *zigzag vi64* is a signed integer encoded as an unsigned `vi64`
by interleaving non-negative and negative values so that
small-magnitude values of either sign occupy small unsigned values,
and thus the shortest `vi64` forms.

For a signed 64-bit integer `n`, the mapping to its unsigned zigzag
representation `z` is:

~~~
encode:  z = (n << 1) ^ (n >> 63)  ; arithmetic right shift
                                   ; equivalently:
                                   ;   n >= 0:  z = 2 * n
                                   ;   n <  0:  z = -2 * n - 1

decode:  n = (z >> 1) ^ -(z & 1)   ; equivalently:
                                   ;   z even:  n =  z / 2
                                   ;   z odd:   n = -(z + 1) / 2
~~~

The first few mappings: 0↔0, -1↔1, 1↔2, -2↔3, 2↔4, -3↔5, 3↔6, ….

The encoded `z` is then serialised as an unsigned `vi64`; decoders
read the `vi64` and apply the decode rule above.

This zigzag mapping is widely used in compact binary serialisation
formats; the description is included here for self-containment.

LOCMAF uses zigzag `vi64` values wherever a signed value is written: for
deltas against the in-group reference, and for the signed list
`trunSampleCompositionTimeOffsets` (ID 5) in both contexts (see
{{parity}}).

# Generic Boxes (genBox) {#genbox}

A `genBox` is the single generic carrier for every box that sits
outside the `moof` in a CMAF chunk — `styp`, `prft`, `emsg`,
`uuid`, and any other ISO box type. One `genBox` element carries
exactly one ISO box.

Boxes *inside* `moof.traf` are not `genBox`es. In particular the
CENC boxes `senc`, `saiz`, and `saio` live inside `traf`, so their
data is carried as `moof`-header fields (`senc`) or recomputed on
reconstruction (`saiz`, `saio`); see {{field-ref}} and {{drm}}.
Only boxes outside the `moof` are `genBox`es.

## Byte layout

A `genBox` element is, in order:

~~~
element_type   vi64      = 1 (genBox)
box_size       vi64      length in bytes of `box_name` + `payload`
box_name       4 bytes   the ISO box type FourCC ('styp','emsg','prft','uuid', ...)
payload        box_size − 4 bytes   the box contents WITHOUT the 8-byte ISO box header
~~~

`element_type` and `box_size` are `vi64` values. `box_size` covers
the `box_name` and the `payload` — the
entire remainder of the element — mirroring ISOBMFF, where a box's
`size` covers its type and contents (the ISO `size` additionally
counts its own 4 bytes). The element on the wire is therefore
`1 | box_size | box_name(4) | payload(box_size − 4)` and is fully
self-delimited by `box_size`; a `box_size` less than 4 is
malformed. Every element that other elements may follow — `genBox`
and the two headers — thus shares the same `type | length | body`
shape, so the length field alone delimits it; only `rawBoxes`,
which nothing ever follows, carries no length ({{rawboxes}}).

The `payload` is the box contents that would follow the 8-byte ISO
box header (`size` + `type`). For a `uuid` box, the 16-byte
`usertype` is part of the box contents: the encoder MUST place the
`usertype` as the first 16 bytes of `payload`, followed by the
remaining box data.

## Full bytes, no delta

`genBox`es always carry full bytes; there is no cross-chunk
`genBox` delta and no `genBox` deletion marker. Presence in the
Object payload means the box is rendered in this chunk, in the
position implied by element order; absence means the box is not
rendered in this chunk. A delta `locmafHeader` MAY still be
combined with full `genBox`es in the same Object — for example a
per-chunk `prft` `genBox` in front of a delta moof header.

## Reconstruction

To reconstruct the ISO box from a `genBox`, the receiver wraps
`payload` in a standard ISO box header:

1. Let `L = 4 + box_size`. `L` MUST fit in 32 bits: a `box_size`
   above `0xFFFFFFFB` is malformed and the receiver MUST reject
   the Object.
2. Emit the box header `uint32be(L) | box_name(4)`, then
   `payload`. The total box is `L` bytes.
3. For a `uuid` box (`box_name == 'uuid'`), the 16-byte `usertype`
   is the first 16 bytes of `payload` (see above) and is therefore
   emitted immediately after the box header as part of `payload`.
   The receiver performs no reordering; reconstruction is the
   standard byte-for-byte wrap.

The ISO `size` escape values 0 (box extends to end of file) and 1
(64-bit `largesize` follows, {{ISOBMFF}}) are never produced: a
reconstructed genBox always carries its actual size in the 32-bit
`size` field. An encoder MUST NOT emit a `genBox` that would
require either escape.

## Ordering

Every `genBox` in the Object payload renders, in payload order,
before the `moof`; the `mdat` immediately follows the `moof`. The
reconstructed chunk is `genBox*` (in payload order), then `moof`,
then `mdat`.

## Box carriage

The `payload` of every `genBox` is simply the box contents as
defined by {{ISOBMFF}} (or the specification owning the box type),
carried verbatim — LOCMAF re-encodes no fields.

The set of box types carried as `genBox`es is open: any box outside
the `moof` is carried under its ISO `box_name` FourCC, which is
self-describing and needs no LOCMAF identifier allocation.

# Raw Boxes (rawBoxes) {#rawboxes}

A `rawBoxes` element carries one or more complete ISO BMFF boxes
verbatim — box headers included — as the entire payload of a LOCMAF
Object. Where a `genBox` carries one pre-`moof` box alongside a
`locmafHeader`, a `rawBoxes` element replaces the header and `mdat`
entirely: it is the escape from the moof-header model for content
that LOCMAF does not otherwise carry. Two uses motivate it:

- **In-band CMAF Header.** In self-framed carriage
  ({{outside-moqt}}), a leading rawBoxes Object holds the `ftyp` +
  `moov` bytes, so a stored LOCMAF segment is self-contained and
  the initialisation bytes round-trip exactly.
- **Verbatim chunk carriage.** A chunk whose `moof` uses structures
  outside the LOCMAF field model rides verbatim, at plain-CMAF
  cost, without forcing the whole track onto plain CMAF packaging.

## Byte layout

A `rawBoxes` element is, in order:

~~~
element_type   vi64                 = 4 (rawBoxes)
boxes          all remaining bytes  complete ISO BMFF boxes, verbatim
~~~

`element_type` is a `vi64`. `boxes` is the concatenation of one or
more complete ISO BMFF boxes {{ISOBMFF}}, each starting with its
own box header, spanning the entire remainder of the Object; the
box sizes MUST sum to exactly that remainder. A `rawBoxes` element
carries no length field of its own: it is always the sole element
of its Object (see below), so the Object length — supplied by MOQT,
or by the `object_length` prefix in self-framed carriage
({{outside-moqt}}) — delimits it, exactly as it delimits the
untagged `mdat` payload of a moof-carrying Object. The ISO `size`
escape values 0 (box extends to end of file) and 1 (64-bit
`largesize` follows) are not allowed: every box in `boxes` MUST
carry its actual size in the 32-bit `size` field. An empty `boxes`
is malformed.

## Sole element of its Object

A `rawBoxes` element MUST be the only element of its LOCMAF Object:
element_type `4` MUST appear first in the Object payload, and a
receiver MUST reject an Object in which it appears after another
element. A rawBoxes Object carries no `genBox`, no `locmafHeader`,
and no `mdat` payload. Restricting rawBoxes to whole Objects keeps
the representation unambiguous — a pre-`moof` box accompanying a
moof header has exactly one carrier, `genBox`, so canonical
comparison ({{canonical}}) never reconciles two encodings of the
same chunk — and is what lets the element drop its length field.

## Delta-state reset

A rawBoxes Object resets the in-group delta chain. On receiving
one, the receiver MUST discard its in-group reference state; the
next moof-header element in the same group MUST be a full header,
and a receiver MUST reject a `locmafDeltaHeader` that follows a
rawBoxes Object without an intervening `locmafFullHeader`. This
rule keeps receivers writer-only: deriving delta state from `boxes`
would require parsing a `moof` out of the raw bytes, which
reconstruction never otherwise needs.

## Reconstruction

The reconstructed bytes of a rawBoxes Object are `boxes`, verbatim.
This is also its canonical form ({{canonical}}): no normalisation
is applied, and canonical comparison is plain byte equality.

# Field Reference {#field-ref}

This section defines the `(field_id, value)` tuples carried inside a
full or delta `locmafHeader` element (element types 2 and 3, see
{{element-sequence}}). These IDs occupy a namespace distinct from
the top-level element-type IDs; they MUST NOT be confused with them.

Framing and value encoding follow the parity rule of {{parity}},
including its four per-field exceptions. LOCMAF governs these IDs
in this document; they are not LOC properties and are not
registered with IANA (see {{iana}}).

The fields are drawn from the boxes inside `moof.traf` — `trun`,
`tfhd`, `tfdt`, and `senc`. Each symbol prefix names its containing
box. The field IDs are identical across full and delta headers;
only the value encoding differs (absolute in a full header, delta
in a delta header — see {{full-chunk}} and {{delta-chunk}}).

| ID | Symbol | Source `moof` field | Kind |
|---:|--------|---------------------|------|
| 1 | `trunSampleSizes` | `trun.sample[i].sample_size` | list |
| 2 | `tfhdSampleDescriptionIndex` | `tfhd.sample_description_index` | scalar |
| 3 | `trunSampleDurations` | `trun.sample[i].sample_duration` | list |
| 4 | `tfhdDefaultSampleDuration` | `tfhd.default_sample_duration` | scalar |
| 5 | `trunSampleCompositionTimeOffsets` | `trun.sample[i].sample_composition_time_offset` | signed list ‡ |
| 6 | `tfhdDefaultSampleSize` | `tfhd.default_sample_size` | scalar |
| 7 | `trunSampleFlags` | `trun.sample[i].sample_flags` | list |
| 8 | `tfhdDefaultSampleFlags` | `tfhd.default_sample_flags` | scalar |
| 9 | `sencInitializationVector` | `senc.sample[i].InitializationVector` | raw bytes |
| 10 | `tfdtBaseMediaDecodeTime` | `tfdt.baseMediaDecodeTime` | scalar |
| 11 | `sencSubsampleCount` | `senc.sample[i].subsample_count` | list |
| 12 | `trunFirstSampleFlags` | `trun.first_sample_flags` | scalar |
| 13 | `sencBytesOfClearData` | `senc.sample[i].subsample[j].BytesOfClearData` | list |
| 14 | `trunSampleCount` | `trun.sample_count` | scalar |
| 15 | `sencBytesOfProtectedData` | `senc.sample[i].subsample[j].BytesOfProtectedData` | list |
| 16 | `sencPerSampleIVSize` | `senc.per_sample_IV_size` | scalar |
| 27 | `deltaDeletedLocmafIDs` | (none — control) | list |

‡ Signed: elements are zigzag `vi64` values (see {{zigzag}}) in
both full and delta context (see {{parity}}).

## Sample flags carry the full 32-bit value

`trunSampleFlags` (ID 7), `tfhdDefaultSampleFlags` (ID 8), and
`trunFirstSampleFlags` (ID 12) each carry the complete 32-bit ISO
`sample_flags` value ({{ISOBMFF}} §8.8.3.1): the full value in a
full header, and a difference in a delta header, both as `vi64`
(the difference signed, see {{zigzag}}).

## Indexing and list lengths

The per-sample list fields (IDs 1, 3, 5, 7, 11) carry the
`sample[i].*` values from their source box. The per-subsample list
fields (IDs 13, 15) carry the
`senc.sample[i].subsample[j].*` values flattened in chunk order,
with a total length equal to the sum of `sencSubsampleCount[i]`
over all samples. `sencInitializationVector` (ID 9) is the
concatenation of per-sample IVs, each `per_sample_IV_size` bytes
long (see {{drm}}).

`trunSampleCount` (ID 14) — always present in a full header, and
carried in a delta header only when it changes — anchors every list
length: the per-sample lists have `trunSampleCount` elements, with
the single exception of `trunSampleSizes` (ID 1), which carries
`trunSampleCount − 1` entries because the last sample size is
derived from the `mdat`-payload length (see
{{sample-size-derivation}}). The receiver therefore knows the
length of every list field before parsing its bytes.

## CENC fields stay inside the header

`sencInitializationVector` (9), `sencSubsampleCount` (11),
`sencBytesOfClearData` (13), `sencBytesOfProtectedData` (15), and
`sencPerSampleIVSize` (16) are drawn from the `senc` box inside
`moof.traf`. They are therefore `locmafHeader` fields, **not**
genBox. Only boxes outside `moof` are carried as generic boxes (see
{{genbox}}). The `saio` and `saiz` boxes are not carried at all;
the receiver recomputes them during canonical reconstruction (see
{{drm}} and {{canonical}}).

## Delta deletion marker {#deletion-marker}

`deltaDeletedLocmafIDs` (ID 27) is a control field used only in a
delta header. It carries the list of field IDs that were present in
the previous chunk of the same group but are absent in the current
one. Its elements are **plain unsigned `vi64` values** — not
zigzag, and
not deltas against any prior deletion list. The list length is
determined by the field's byte-length prefix: the receiver reads
unsigned `vi64` values until the prefix is exhausted. This field
MUST NOT
appear in a full header. Its application is specified in
{{delta-chunk}}.

# Full Chunk Encoding {#full-chunk}

A full `locmafHeader` (element type 2) carries an absolute encoding
of one CMAF chunk's `moof`. Any boxes that preceded the `moof` in
the source chunk (`styp`, `prft`, `emsg`, …) are carried as genBox
elements ahead of the header (see {{genbox}}); the `mdat` payload
follows the header's property block unchanged.

In a full header, values are absolute, encoded per {{parity}}. The
first moof-carrying Object of every MOQT group MUST carry a full
header so a subscriber tuning in at a group boundary has a
complete in-group reference (see {{element-sequence}}).

## Emission rules

The encoder first derives each sample's *effective* duration,
flags, and composition-time offset from the source `moof` — the
per-sample `trun` value when present, else the `tfhd` default when
present, else the `trex` default (offsets default to 0) — and emits
fields from those effective values. The encoding is therefore
independent of how the source `moof` happened to distribute values
between per-sample entries and defaults: two decode-equivalent
source `moof`s yield the same emitted fields.

{: newline="true"}
`trunSampleCount` (14)
: always.

`tfdtBaseMediaDecodeTime` (10)
: always.

`tfhdSampleDescriptionIndex` (2)
: emit iff the effective sample-description index ≠ `trex.default_sample_description_index`.

`trunSampleDurations` (3)
: emit iff the effective sample durations are not all equal.

`tfhdDefaultSampleDuration` (4)
: emit iff the effective sample durations are all equal AND that value ≠ `trex.default_sample_duration`.

`trunSampleSizes` (1)
: emit iff sample sizes are not all equal AND `sample_count > 1`; the list carries `sample_count − 1` values (the first `n − 1` in chunk order).

`tfhdDefaultSampleSize` (6)
: emit iff all samples in the chunk share one size AND that size ≠ `trex.default_sample_size` AND `sample_count > 1` (see {{sample-size-derivation}}).

`trunSampleFlags` (7)
: emit iff the effective sample flags are neither all equal, nor equal on all samples but the first.

`trunFirstSampleFlags` (12)
: emit iff `sample_count > 1` AND the first sample's effective flags differ from the (equal) effective flags of all other samples.

`tfhdDefaultSampleFlags` (8)
: emit iff the effective flags shared by the samples it covers (all samples, or all but the first when `trunFirstSampleFlags` is emitted) ≠ `trex.default_sample_flags`; never emitted together with `trunSampleFlags`.

`trunSampleCompositionTimeOffsets` (5)
: emit iff any effective composition-time offset ≠ 0.

`sencPerSampleIVSize` (16)
: emit iff `senc` is present AND `per_sample_IV_size ≠ tenc.default_Per_Sample_IV_Size`.

`sencInitializationVector` (9)
: emit iff `senc` is present AND `per_sample_IV_size > 0`.

`sencSubsampleCount` (11), `sencBytesOfClearData` (13), `sencBytesOfProtectedData` (15)
: emit iff `senc` is present AND the samples carry subsample maps.

These rules produce the minimal encoding and are part of the
canonical encoding ({{canonical-encoding}}). An encoder MAY
additionally emit a field whose value matches the applicable
default — strict `cmf2` mode ({{scope}}) does exactly that for the
four `tfhd` defaults — without affecting the decoded effective
values or the canonical reconstruction ({{canonical}}).

## Sample-size derivation {#sample-size-derivation}

Let `n = trunSampleCount` and let `P` be the chunk's `mdat`-payload
length (the MOQT object length minus the bytes consumed by all
elements). The receiver MUST derive sample sizes as follows:

- If `trunSampleSizes` (ID 1) is present, it carries exactly
  `n − 1` values. `sample_size[i] = listed[i]` for `i` in
  `[0, n−1)`, and `sample_size[n−1] = P − sum(listed)`. The
  receiver MUST NOT consult `tfhdDefaultSampleSize`,
  `trex.default_sample_size`, or any other source for sample sizes
  in this chunk.
- Else if `tfhdDefaultSampleSize` (ID 6) is present, all `n`
  samples have that size.
- Else if `n == 1`, the lone sample's size is `P`. The encoder
  omits all size fields for a single-sample chunk (see below), so
  the payload length is authoritative and
  `trex.default_sample_size` is never consulted at `n == 1`.
- Else if `trex.default_sample_size` is non-zero, all `n` samples
  have that size.
- Else if `P == 0`, all `n` samples have size 0 (e.g. an event
  track carrying several zero-size samples per chunk).
- Else the chunk is malformed and the receiver MUST reject it.

Correspondingly, when `sample_count == 1` both `trunSampleSizes`
and `tfhdDefaultSampleSize` MUST be omitted — the single sample's
size is always `P`. When `sample_count > 1` with uniform sizes the
encoder MUST emit `tfhdDefaultSampleSize` (subject to the
`trex.default_sample_size` equality rule) and MUST NOT emit
`trunSampleSizes`; when sizes vary the encoder MUST emit
`trunSampleSizes` with exactly `n − 1` entries and MUST NOT emit
`tfhdDefaultSampleSize`. The receiver MUST reject a chunk in which
`sum(listed) > P`, a chunk whose default-derived sizes do not
satisfy `n × size = P`, and a chunk with `n == 0` whose `mdat`
payload is not empty. Omitting the last sample size shaves one `vi64`
per chunk; using the default for uniform-size tracks (common for
fixed-bitrate audio, e.g. AC-3) collapses `n − 1` `vi64` values into
one.

# Delta Chunk Encoding {#delta-chunk}

A delta `locmafHeader` (element type 3) carries only the
differences between the current chunk's `moof` and the most
recently received full chunk in the same MOQT group. Delta encoding
is additive: a field absent from the delta header is unchanged from
the previous chunk.

## Field value encoding

Each emitted field's value is interpreted relative to its kind:

| Kind | Wire encoding | Reconstruction |
|------|---------------|----------------|
| scalar (even ID) | zigzag `vi64` (see {{zigzag}}) of `current − previous` | `current = previous + delta` |
| list (odd ID) | zigzag `vi64` per element, concatenated; element delta = `current[i] − previous[i]` | element-wise sum with the previous list |
| raw bytes (odd ID) | full new bytes, verbatim | overwrite the previous bytes |

The "previous value" for each field is the effective value used in
the reconstruction of the previous LOCMAF chunk in the same group
(or, after a mid-group full header, the previous chunk starting
from that re-anchor).

### List length changes

The length of each per-sample list in the current chunk is the
chunk's effective `trunSampleCount`: the value established by the
in-group reference, changed only when a delta for ID 14 is present
(absence means unchanged, like any delta field).
This covers `trunSampleDurations` (3),
`trunSampleCompositionTimeOffsets` (5), `trunSampleFlags` (7), and
`sencSubsampleCount` (11). `trunSampleSizes` (1) carries
`trunSampleCount − 1` entries (see {{sample-size-derivation}}). The
per-subsample lists (13, 15) have a total length equal to the sum
of the new `sencSubsampleCount[i]`. Consequently the receiver knows
`len(current)` for every list before parsing its bytes.

When `len(current) ≠ len(previous)` the delta rule extends as
follows:

- For indices `i` in `[0, min(len(current), len(previous)))`: the
  wire carries `zigzag(current[i] − previous[i])` and the receiver
  reconstructs `current[i] = previous[i] + delta[i]`.
- For indices `i` in `[len(previous), len(current))` (current
  longer): the wire carries `zigzag(current[i])`, the absolute
  value, equivalent to treating the missing previous entry as 0.
  The receiver reconstructs `current[i] = delta[i]`.
- For indices `i` in `[len(current), len(previous))` (current
  shorter): no bytes are emitted for these positions; the receiver
  truncates to `len(current)`.

## `tfdt` BMDT derivation {#bmdt-derivation}

The receiver derives a delta chunk's BMDT as `previous_bmdt`
plus the sum of the previous chunk's *effective* sample durations
(the per-sample values when present, else `sample_count` times the
applicable default). `tfdtBaseMediaDecodeTime` (ID 10) is a
full-header field: it MUST NOT appear in a delta header, and a
receiver MUST reject a delta header that carries it. The
derivation is safe because CMAF ({{CMAF}}) requires the decode
timeline to be contiguous: each fragment's `baseMediaDecodeTime`
equals the previous fragment's plus the sum of its sample
durations.

When the source BMDT diverges from this derivation (a splice, a
capture gap, a stream re-anchor), the encoder MUST emit a full
LOCMAF chunk ({{full-chunk}}): a timeline discontinuity re-anchors
the entire in-group reference, which is also what a recovering or
late-joining receiver needs at exactly that point.

## Deletions {#deletions}

Because delta encoding is additive, a field that was present in the
previous chunk but is genuinely gone from the current chunk cannot
be signalled by mere absence — absence means "unchanged." The
deletion marker `deltaDeletedLocmafIDs` (ID 27, see
{{deletion-marker}}) provides that signal: it lists the field IDs
present in the previous chunk that no longer apply. The receiver
MUST apply deletions **before** applying deltas, removing each
listed field from the previous-chunk state so the current chunk
falls back to the appropriate default (`trex`-derived or absent).

The motivating case is `trunFirstSampleFlags` (ID 12). A SAP-1
random-access chunk emits this field to flag its first sample as a
sync sample; the immediately following non-sync chunk must say
"this override no longer applies" so the receiver falls back to
`trex.default_sample_flags` for the first sample. The second chunk
emits ID 27 with a one-element list containing the value 12. The
typical cost is two bytes — one length-prefixed list of one field
ID — versus the tens to hundreds of bytes of re-anchoring with a
full header. In a one-sample-per-chunk stream the same pattern
rides `tfhdDefaultSampleFlags` (ID 8) instead: the sync chunk
emits ID 8 and the next chunk deletes it, falling back to
`trex.default_sample_flags`.

## Empty delta

An empty delta property block (`properties_length == 0`) is valid
and means "no field changed since the previous chunk." This is the
steady-state case for sample-level fragmented streams. The on-wire
object reduces to the delta header element plus the `mdat` payload.

# DRM and Common Encryption {#drm}

LOCMAF preserves the per-sample Common Encryption (CENC) {{CENC}}
metadata that EME-based decryption pipelines require. The track's
key identifier, scheme, pattern parameters, and other static
encryption defaults are carried in the CMAF Header's
`tenc` box (inside `schi` inside `sinf`) and signalled in the
catalog through CMSF `contentProtections` and
`contentProtectionRefIDs` (see {{catalog}}). The per-sample
material that varies chunk to chunk is carried as `locmafHeader`
fields.

## Protection schemes

LOCMAF is scheme-agnostic. The packaging carries the per-sample
`senc` metadata — initialization vectors and subsample maps — for
any CENC {{CENC}} protection scheme; the scheme itself is
identified by `tenc.default_isProtected = 1` and the four-character
`scheme_type` of the surrounding `schm` box, both carried in the
CMAF Header, and the reconstruction of {{canonical-cenc}} does not
depend on it. Schemes with per-sample initialization vectors
(`cenc`, `cbc1`, `cens`) carry them explicitly in
`sencInitializationVector` (ID 9) on every chunk; LOCMAF defines
no IV derivation or prediction rule.

One scheme has scheme-specific behaviour. Under `cbcs` (AES-128-CBC
subsample pattern encryption), the constant initialization vector
(`tenc.default_constant_IV`) and the pattern
(`default_crypt_byte_block` / `default_skip_byte_block`) travel in
the CMAF Header ({{CENC}} §10.4). No per-sample IV appears in
`senc`: `per_sample_IV_size` is 0, `sencInitializationVector`
(ID 9) is empty and `sencPerSampleIVSize` (ID 16) is 0 or omitted;
the reconstructed `senc` still sets `flags = 0x000002` and carries
the subsample map.

In practice, CMAF presentation profiles ({{CMAF}} Annex A) and the
deployed DRM ecosystem {{DASHIF-ECCP}} use `cenc` and `cbcs`.

## Box handling

| Box | Where in CMAF | LOCMAF treatment |
|-----|---------------|------------------|
| `senc` | inside `traf` | per-sample IVs and subsample maps carried via `locmafHeader` field IDs 9, 11, 13, 15, 16. |
| `saio` | inside `traf` | not carried; recomputed by the receiver to point at the reconstructed `senc` (see {{canonical}}). |
| `saiz` | inside `traf` | not carried; recomputed from the per-sample IV size and subsample counts (see {{canonical}}). |
| `tenc` | inside `schi` in `moov` | carried verbatim in the CMAF Header. |

The following DRM boxes are not supported. Sources that require
them MUST use plain CMAF packaging:

| Box | Reason for exclusion |
|-----|----------------------|
| `sgpd` / `sbgp` | Mid-fragment key rotation via `seig` sample groups is out of scope; KID changes MUST align with fragment boundaries (see {{scope}}). |
| `pssh` (per-fragment) | License-acquisition information is signalled via the CMSF `contentProtections` mechanism (see {{catalog}}), per {{CMAF}} §7.4.3. |
| `subs` | Sub-sample information for image subtitle profiles (e.g. `im1i`) is out of scope. |

# Event-Only Tracks {#event-only}

DASH-IF Ingest {{DASH-IF-INGEST}} defines a CMAF-based push
interface for live encoders. One of its track shapes is the sparse
event-only track: a CMAF track that carries no media samples (or
zero-size samples) and exists purely to deliver timed events via
`emsg` boxes attached to its chunks. In LOCMAF, each such `emsg`
box rides verbatim as a genBox element ahead of the chunk's
`locmafHeader` (see {{genbox}}). Multiple events in one chunk
become multiple genBox elements.

LOCMAF supports event-only tracks with no further extension. A full
header for an event-only group sets `trunSampleCount = 0`, carries
`tfdtBaseMediaDecodeTime`, is preceded by the chunk's `emsg`
genBoxes, and is followed by an empty `mdat` payload (the receiver
reconstructs an `mdat` box with an 8-byte header only). Two
encoder strategies are valid for subsequent chunks in the same
group:

1. **Full header per chunk.** Every event chunk carries a full
   header with `trunSampleCount = 0` and its absolute
   `tfdtBaseMediaDecodeTime`, since a zero sample count produces
   no derivation increment ({{bmdt-derivation}}). This costs a few
   bytes per chunk and is recommended for sparse event-only
   tracks.
2. **Synthetic per-chunk sample.** The encoder sets
   `trunSampleCount = 1` with a `default_sample_duration` equal to
   the intended per-chunk advancement and a zero-size sample. BMDT
   derivation then works, and subsequent chunks use delta headers.
   This matches how DASH-IF Ingest commonly shapes sparse metadata
   tracks (`urim`, `stpp`).

For new MOQT deployments, an MSF `eventtimeline` companion track
{{MSF}} is the preferred mechanism for event metadata. LOCMAF
event-only tracks are intended for gateways that transit-relay CMAF
Ingest content unchanged across MOQT.

# Canonical Reconstruction {#canonical}

This section defines a deterministic CMAF rebuild. Given the same
effective values ({{effective-values}}), genBox list, `mdat`
payload, and CMAF Header, every conformant implementation that
follows this section produces **byte-identical** output. That
byte-identical output is the *canonical form* and is the reference
used for conformance and golden-vector comparison.

A rawBoxes Object ({{rawboxes}}) needs no rebuild: its canonical
form is its `boxes` bytes, verbatim. The remainder of this section
applies to moof-carrying Objects.

Because the canonical form is a function of the effective values
alone, it is invariant to every encoder representation choice:
where the group was split into full and delta chunks, field
ordering, redundant fields such as strict-`cmf2` `tfhd` defaults,
and whether a value travelled as a per-sample list or as a default.
Two LOCMAF encodings decode to the same canonical bytes iff they
carry the same media — which is exactly the comparison an interop
harness needs.

A decoder MAY emit any functionally-equivalent CMAF that suits its
own pipeline — a different legal box order, a different `tr_flags`
packing, or extra `tfhd` defaults are all acceptable for local
playback. The canonical form is not a constraint on what a decoder
feeds its renderer; it is the single byte-comparable reference an
interop harness regenerates and diffs. A receiver MUST NOT assume
that some other endpoint's reconstructed bytes match its own unless
both produced the canonical form.

## Effective values {#effective-values}

Decoding a chunk — applying deltas and deletions to the in-group
reference, then the derivations of {{sample-size-derivation}} and
{{bmdt-derivation}} — yields the chunk's *effective values*:

- `n = trunSampleCount` and the BMDT;
- the effective sample-description index: field 2 when present,
  else `trex.default_sample_description_index`;
- for each sample `i` in `[0, n)`:
  - `duration[i]`: field 3's element `i` when present, else field 4
    when present, else `trex.default_sample_duration`;
  - `size[i]`: per {{sample-size-derivation}};
  - `flags[i]`: field 7's element `i` when present; else field 12
    when present and `i == 0`; else field 8 when present; else
    `trex.default_sample_flags`;
  - `cto[i]`: field 5's element `i` when present, else 0;
  - when CENC is in use: `IV[i]` (field 9) and the subsample map
    (fields 11, 13, 15);
- the ordered genBox list (see {{genbox}}) and the `mdat` payload
  `P`, whose length is the MOQT object length minus the bytes
  consumed by all preceding elements.

The effective values are the chunk's meaning: they are what a
frame interface consumes directly ({{consumption}}), and the only
chunk-derived input to the canonical reconstruction below. The
remaining inputs come from the CMAF Header `moov`: the single
`trak`'s `track_ID`, the `trex` defaults, `mdhd.timescale`, and
(when CENC is in use) the `tenc` defaults.

## Chunk box order

The reconstructed chunk is the concatenation, in this order:

1. each genBox, wrapped per {{genbox}}, in payload order;
2. the `moof` box;
3. the `mdat` box.

Inside `moof`, the order is `mfhd` then `traf`. Inside `traf` the
order is `tfhd`, `tfdt`, `trun`, and — when CENC metadata is present
— `saiz`, `saio`, `senc` appended in that order. This `saiz`/`saio`/
`senc` order is fixed by this section (see {{canonical-cenc}}).

## mfhd

`version=0`, `flags=0`. LOCMAF does not carry
`mfhd.sequence_number` and it is not reconstructable from the LOCMAF
object alone; `mfhd.sequence_number` is not load-bearing for CMAF
chunk decode in playback pipelines. The canonical rule is therefore
`mfhd.sequence_number = 0`. (An implementation that needs the real
sequence number can derive it from the MOQT object identity, but
canonical comparison uses 0.)

## tfhd {#canonical-tfhd}

`version=0`. The canonical `tf_flags`:

- MUST set `default-base-is-moof` (`0x020000`);
- MUST NOT set `base-data-offset-present` (`0x000001`);

so sample-data offsets are relative to the start of the containing
`moof` ({{ISOBMFF}}). Each optional default sets its flag and emits
its value iff the effective values call for it — wire presence is
irrelevant, so a strict-`cmf2` encoding ({{scope}}) canonicalises
identically to its minimal counterpart:

- `sample-description-index-present` (`0x000002`) iff the effective
  sample-description index ≠
  `trex.default_sample_description_index`;
- `default-sample-duration-present` (`0x000008`) iff the effective
  durations are all equal AND that value ≠
  `trex.default_sample_duration`;
- `default-sample-size-present` (`0x000010`) iff the uniform-size
  case of {{canonical-sizes}} places a size here;
- `default-sample-flags-present` (`0x000020`) iff the effective
  flags covered by the default (all samples, or all but the first
  when first-sample-flags is used, see {{canonical-trun}}) are
  equal AND ≠ `trex.default_sample_flags`.

`tfhd.track_ID` is set to the `track_ID` of the single `trak` in
the CMAF Header's `moov`. The field order in the box is the
standard ISO order: `track_ID`, then each present optional in
flag-bit order.

## tfdt

`version=1` always (64-bit `baseMediaDecodeTime`), `flags=0`. The
value is the absolute BMDT — carried in a full header, or derived
per {{bmdt-derivation}} for a delta chunk. A fixed
version avoids a magnitude-dependent conditional in the canonical
form, and live decode timelines routinely exceed 32 bits anyway
(2^32 ticks is under 14 hours at a 90 kHz timescale, and
wallclock-anchored timelines are past it from the start); the 4
extra bytes exist only in the reconstructed chunk, not on the wire.

## trun {#canonical-trun}

`trun.version = 1` iff any effective composition-time offset is
negative, otherwise `0`.

The canonical `tr_flags`, set and emitted in this order:

- `data-offset-present` (`0x000001`) — always set;
- `first-sample-flags-present` (`0x000004`) — set iff `n > 1` and
  the first sample's effective flags differ from the (equal)
  effective flags of all other samples;
- `sample-duration-present` (`0x000100`) — set iff the effective
  durations are not all equal;
- `sample-size-present` (`0x000200`) — set iff per-sample sizes are
  used (see {{canonical-sizes}});
- `sample-flags-present` (`0x000400`) — set iff the effective flags
  are neither all equal nor covered by the first-sample-flags case
  above;
- `sample-composition-time-offsets-present` (`0x000800`) — set iff
  any effective composition-time offset ≠ 0.

The `trun` field layout is `sample_count`, then `data_offset`, then
(if `first-sample-flags-present`) `first_sample_flags` (which is
`flags[0]`), then the per-sample records populated from the
effective vectors, each carrying its present fields in ISO order:
`sample_duration`, `sample_size`, `sample_flags`,
`sample_composition_time_offset`. When `trun.version == 1` the
composition-time offset is a signed 32-bit value; when
`trun.version == 0` it is unsigned (reached only when every offset
is ≥ 0).

When `n == 0` (an event-only chunk, {{event-only}}) there are no
effective per-sample values: every conditional `tr_flags` bit above
is clear, no per-sample records are emitted, and no optional `tfhd`
default is present ({{canonical-tfhd}}).

### Canonical sample-size layout {#canonical-sizes}

Let `n = trunSampleCount` and `P = len(mdat payload)`.

- **Uniform sizes** (all `n` samples share a single size `s`; a
  single-sample chunk is trivially uniform with `s = P`). Place `s`
  in `tfhd.default_sample_size` and set its flag per
  {{canonical-tfhd}} iff `s ≠ trex.default_sample_size`; when `s`
  equals `trex.default_sample_size`, omit it from `tfhd` and rely
  on `trex`. Emit no per-sample sizes in `trun` and clear
  `sample-size-present`. Note that while the wire omits a single
  sample's size ({{sample-size-derivation}}), the canonical CMAF
  chunk MUST still carry it whenever it differs from the `trex`
  default: ISO BMFF has no rule deriving a sample size from the
  `mdat` length, so a chunk without it would not be
  decode-equivalent.
- **Varying sizes** (`n > 1`, sizes differ). Emit per-sample sizes
  in `trun` and set `sample-size-present`; do **not** set
  `tfhd.default_sample_size`. On the wire LOCMAF carries `n − 1`
  sizes (field 1); the canonical `trun` reconstructs all `n` with
  `sample_size[n−1] = P − sum(first n−1 sizes)`.

Malformed cases the receiver MUST reject:

- `n > 1` with non-uniform sizes and no per-sample size list;
- `sum(listed sizes) > P`;
- sizes derived from a default with `n × size ≠ P`;
- `n == 0` with a non-empty `mdat` payload.

## data_offset and the mdat header {#canonical-mdat}

`trun.data_offset = moof_size + 8`: it points at the first byte of
the `mdat` sample data, just past the 8-byte `mdat` header, with
`default-base-is-moof` making offsets relative to the `moof` start.
The `mdat` header is always 8 bytes: `uint32be(8 + len(P)) |
'mdat'`. The ISO `size` escapes 0 and 1 are not allowed here
either: an `mdat` payload that does not fit the 32-bit size
(`len(P) > 0xFFFFFFF7`) cannot be packaged as LOCMAF, and a
receiver MUST reject a chunk that would require it.

## CENC senc / saiz / saio reconstruction {#canonical-cenc}

This section applies when the track is protected
(`tenc.default_isProtected = 1`) and the chunk's effective values
({{effective-values}}) include per-sample auxiliary information: a
non-zero `per_sample_IV_size`, an effective subsample map, or
both. Wire presence in the current header is irrelevant — under
delta encoding every CENC field may be inherited from the in-group
reference. The receiver reconstructs `senc`, `saiz`, and `saio`
with the following pinned layouts; `saiz` and `saio` are never
carried on the wire (see {{drm}}) and are recomputed here.

When a protected chunk has no per-sample auxiliary information —
`per_sample_IV_size` is 0 and there is no subsample map, as under
`cbcs` full-sample encryption with a constant IV — none of the
three boxes is emitted.

**`senc`.** `version=0`. `flags = 0x000002`
(`senc_use_subsamples`) iff the effective subsample map is
present, otherwise `flags = 0`. Layout: `sample_count = n`, then
for each sample `i`:

- the `InitializationVector`, `per_sample_IV_size` bytes, taken
  from `sencInitializationVector` (field 9);
- when `flags = 0x000002`: `subsample_count[i]` as `uint16` (from
  field 11; 0 for a sample without subsamples), followed by
  `subsample_count[i]` pairs of `(BytesOfClearData` as `uint16`,
  `BytesOfProtectedData` as `uint32)` drawn from fields 13 and 15,
  flattened in chunk order. Every sample carries the
  `subsample_count` field when the flag is set.

**`saiz`.** `version=0`, `flags=0`; with `flags=0` the optional
`aux_info_type` and `aux_info_type_parameter` fields are omitted —
their defaults per {{CENC}} §7.1 are the track's protection scheme
and 0. For each sample, the auxiliary information size is:

~~~
aux_size[i] = per_sample_IV_size               ; flags = 0
aux_size[i] = per_sample_IV_size
            + 2 + 6 * subsample_count[i]       ; flags = 0x000002
~~~

so when the subsample flag is set, a sample with
`subsample_count = 0` still counts its 2-byte `subsample_count`
field, matching the `senc` layout above. An `aux_size[i]` above
255 does not fit the 8-bit `sample_info_size` of {{ISOBMFF}} and
the receiver MUST reject the chunk. If all `aux_size[i]` are
equal, set `default_sample_info_size` to that common value,
`sample_count = n`, and emit an empty per-sample array. Otherwise
set `default_sample_info_size = 0`, `sample_count = n`, and emit
the `n`-entry `sample_info_size` array.

**`saio`.** `version=0`, `flags=0`, `entry_count=1`, with a single
`offset` locating the first byte of the first sample's auxiliary
information in `senc` (the first per-sample IV when
`per_sample_IV_size > 0`). Per
{{ISOBMFF}}, `saio` offsets in a movie fragment are relative to the
same base as `trun.data_offset` — with `default-base-is-moof`
({{canonical-tfhd}}), the first byte of the `moof`. genBoxes
preceding the `moof` therefore do not enter into the offset:

~~~
saio.offset = offset_of_senc_within_moof + 16
~~~

where `offset_of_senc_within_moof` is the byte offset of the `senc`
box from the start of the `moof`, and the 16 bytes skip the `senc`
box header (8 bytes), the `FullBox` version and flags (4 bytes),
and the `sample_count` field (4 bytes). The 32-bit offset of
`version=0` always suffices, since the offset points within the
`moof`, whose size is bounded by its own 32-bit box size.

## Canonical encoding {#canonical-encoding}

The encode side has an equally deterministic reference form. A
LOCMAF encoding is *canonical* when:

- every full and delta header emits exactly the fields required by
  the emission rules of {{full-chunk}}, with no redundant fields;
- every delta header carries exactly the fields whose effective
  values changed from the in-group reference state, plus
  `deltaDeletedLocmafIDs` listing exactly the fields that leave
  that state ({{deletions}});
- a full header appears only as the first moof-carrying Object of
  each group, immediately after a rawBoxes Object ({{rawboxes}}),
  or where {{bmdt-derivation}} requires one;
- field IDs appear in ascending order; and
- every `vi64` uses its shortest form.

Two canonical encoders given the same CMAF input therefore produce
byte-identical LOCMAF Objects, enabling golden vectors on the
encode side as well as the decode side. Canonical encoding is not
required for interoperability: any conformant encoding decodes to
the same effective values and thus the same canonical CMAF.

## Implementation note

Canonical reconstruction requires a normalisation pass: an
implementation MUST NOT rely on incidental serialiser output (e.g.
box order or `tr_flags` packing produced by a general-purpose ISO
BMFF writer) to match the canonical bytes. Golden vectors compare
the reconstructed genBox and `moof` bytes exactly; the `mdat`
payload is passed through unchanged.

# Consumption: Chunk and Frame Interfaces {#consumption}

LOCMAF carries the same elementary encoded samples as LOC
{{LOC}}: the `mdat` payload is the concatenation of coded samples
in decode order, and the moof-header fields give each sample's
size, decode time, composition-time offset, and (when protected)
its CENC metadata. A receiver therefore has two equally valid
consumption paths, matching the two shapes playback interfaces
come in: those that accept ISO BMFF chunks and those that accept
individual media frames.

- **Chunk interfaces.** Reconstruct a CMAF chunk per {{canonical}}
  and feed it to an interface that accepts ISO BMFF / CMAF input,
  initialised with the CMAF Header (see {{cmaf-header}}). In the
  browser this is Media Source Extensions (MSE): append the chunk
  to a `SourceBuffer` whose initialisation segment is the CMAF
  Header. Native media pipelines built around a fragmented-MP4
  demuxer consume the same reconstructed chunks.
- **Frame interfaces.** Slice the `mdat` payload into per-sample
  byte ranges using the effective sample values
  ({{effective-values}}) and feed each coded frame to a decoder,
  using the decode time, composition-time offset, and sample flags
  to set the frame's timestamp, duration, and key/delta type. The
  CMAF Header supplies the codec configuration, and for
  CENC-protected content the effective per-sample IV and subsample
  map supply the decryption parameters. In the browser this is
  WebCodecs: each frame becomes an `EncodedVideoChunk` /
  `EncodedAudioChunk` fed to a `VideoDecoder` / `AudioDecoder`.
  Native decoder APIs that accept individual coded frames work the
  same way.

LOCMAF is a media container, not tied to any particular playback
interface; neither path is privileged by this document. Producing
the canonical CMAF chunk is required only for golden-vector
conformance, not for playback.

# Use Outside MOQT {#outside-moqt}

This section is informative. LOCMAF is specified for carriage as
MOQT Object payloads, where the transport supplies the framing:
MOQT gives every Object its length, the subgroup gives in-order
delivery, and the group boundary marks where a full header
re-anchors the delta chain. A LOCMAF Object carries no length of
its own, so any use outside MOQT has to restore that framing. This
section sketches the minimal wrapping; a full specification of
such carriage is out of scope for this document.

A *LOCMAF segment* is the self-framed equivalent of one MOQT
group: the concatenation of the group's LOCMAF Objects
({{object-encoding}}) in decode order, each prefixed by its
length —

~~~
object_length   vi64
LOCMAF Object   object_length bytes
~~~

repeated once per Object. The first moof-carrying Object of a
segment carries a full header, exactly as at a MOQT group boundary,
and delta chunks reference the preceding Object in the same segment
({{delta-chunk}}).

The same framing doubles as a storage format. A LOCMAF segment is
directly a file on disk, and segments concatenate into longer
files that remain parseable, since every Object is length-prefixed
and every full header re-anchors decoding. The CMAF Header is
either stored alongside, as for CMAF initialisation and media
segments, or carried in-band as a leading length-prefixed rawBoxes
Object ({{rawboxes}}) holding the `ftyp` + `moov` bytes verbatim.
The in-band form makes the file self-contained: a reader obtains
the initialisation bytes exactly as published and decodes every
subsequent Object against them, just as over MOQT.

The framing also maps directly onto low-latency HTTP delivery:

- In low-latency DASH, a LOCMAF segment is delivered progressively
  under chunked transfer encoding, each length-prefixed Object
  taking the role a CMAF chunk has today, with the CMAF Header
  referenced from the MPD as the initialisation segment.
- In low-latency HLS, each part carries one or more
  length-prefixed Objects. A part advertised as independent begins
  with an Object carrying a full header, so that a client joining
  there has a complete decode anchor — the part-level counterpart
  of the mid-group full header of {{element-sequence}}. Because
  the framing is part of the payload rather than of the HTTP
  envelope, concatenating the parts of a segment yields the same
  bytes as the DASH segment and the same file as stored at rest.

The catalog signalling of {{catalog}} has no MPD or playlist
counterpart defined here: at minimum the packaging, the
`locmafVersion`, the initialisation-data reference, and any
content-protection signalling need manifest-level equivalents, and
the segments need a dedicated media type (such as `video/locmaf`,
`audio/locmaf`, or `application/locmaf`) so that clients do not
mistake them for CMAF. Registration of such media types would
accompany a future specification of this carriage.

# Security Considerations

LOCMAF is a compact packaging for CMAF media and introduces no new
authentication or confidentiality mechanism. It is carried as MOQT
{{MOQT}} Object payloads and inherits QUIC's transport security.
Per-sample encryption metadata defined by {{CENC}} is preserved
through the LOCMAF round-trip; LOCMAF neither weakens nor
strengthens the underlying DRM scheme.

A receiver expands attacker-influenceable input (the delta state,
the per-sample lists, the genBox payloads) into ISO BMFF
{{ISOBMFF}} boxes that are then fed to a media pipeline. A receiver
MUST validate that every reconstructed box is well-formed before
passing it onward, and in particular:

- The receiver MUST bound the length of every reconstructed
  per-sample and per-subsample list against `trunSampleCount` (and,
  for the subsample lists, against the reconstructed
  `sencSubsampleCount` totals) before allocating or copying.
- The receiver MUST verify that the sum of reconstructed
  `sencBytesOfClearData` and `sencBytesOfProtectedData` for each
  sample equals that sample's size.
- The receiver MUST verify that every reconstructed
  `BytesOfClearData` value fits in 16 bits and every
  `BytesOfProtectedData` value fits in 32 bits — the `senc` field
  widths of {{canonical-cenc}} — and MUST reject the chunk
  otherwise.
- The receiver MUST verify that every sample's reconstructed
  auxiliary-information size fits the 8-bit `saiz`
  `sample_info_size` ({{canonical-cenc}}) and MUST reject the
  chunk otherwise.
- The receiver MUST reject the malformed sample-size cases listed
  in {{canonical-sizes}}.
- The receiver MUST treat an unknown leading element_type as a
  malformed object (see {{element-sequence}}): unknown top-level
  element types are not self-delimiting and MUST NOT be skipped.

Each genBox payload is wrapped into an ISO BMFF box of declared
length `4 + box_size`; the receiver MUST ensure `box_size` is at
least 4, at most `0xFFFFFFFB`, and does not exceed the bytes
actually present in the element, and MUST validate the wrapped box
against the structural rules for its `box_name` before use.

A rawBoxes element's `boxes` content ({{rawboxes}}) is likewise
attacker-influenceable and is passed onward verbatim. Before use,
the receiver MUST verify that it parses as a sequence of complete
top-level boxes — each declared `size` at least 8, neither ISO
`size` escape used, and the sizes summing to exactly the Object
bytes that follow the element_type — and MUST validate each box
against the structural rules for its type.

Replay considerations within a MOQT group are inherited from
{{MOQT}}; LOCMAF adds no new replay attack surface.

# IANA Considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The initial version of LOCMAF was developed as part of the Master
Thesis work of Hugo Björs at Eyevinn Technology, supervised by
Torbjörn Einarsson.

The author thanks the Media over QUIC working group, in particular
the authors and contributors to {{MOQT}}, {{MSF}}, and {{CMSF}},
for the prior art this work builds on.
