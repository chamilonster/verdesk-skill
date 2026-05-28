# Specification

**Title of the invention.** Grid-Based Modulated Visual Pipeline with
Deterministic Perceptual-Hash Cell Classification for AI Agents.

---

## Cross-Reference to Related Applications

None.

## Statement Regarding Federally Sponsored Research or Development

None. The invention was developed without funding, contract, or grant from
any United States government agency.

---

## 1. Field of the Invention

The present invention relates generally to the interaction between
computational systems and external artificial-intelligence (AI) agents, and
more particularly to methods, systems, and machine-readable media for
capturing, modulating, classifying, and transmitting visual and structural
content of a visual source to such an AI agent with substantially reduced
consumption of the vision-related computational resources thereof. The
visual source may include, without limitation, the display of a host
computer, a webcam or other locally attached image sensor, a network
camera, a video surveillance feed, a robotic vision system, a vehicle
camera, a medical or scientific imaging device, an unmanned aerial vehicle
camera, or any other source producing a time-varying sequence of images.

## 2. Background of the Invention

Modern AI agents that interact with graphical user interfaces — including
those marketed under names such as Computer Use, Copilot Vision, and
several browser-based automation frameworks — consume a substantial portion
of their inference budget receiving and re-receiving complete screenshots of
the host computer's display. On every interaction cycle, the entire screen
is captured, encoded as a raster image, transmitted to the model, and
re-interpreted from scratch.

This approach has at least four shortcomings: (a) the cost of vision tokens
grows linearly with session length, even when the underlying display has
changed only marginally; (b) the latency added by transmitting and
interpreting the full frame on each cycle is non-negligible; (c) information
that is already available to the host computer in structured form, such as
plain text rendered on the screen or the underlying accessibility tree, is
forced through the visual pathway at vision-token cost; and (d) for accessibility-oriented use cases, latency and cost together render full-frame
agents impractical for users who require continuous assistance.

Similar inefficiencies affect AI agents that consume video streams from
sources other than computer displays. Video surveillance systems, robotic
vision pipelines, machine-vision applications in industrial settings,
autonomous-vehicle perception stacks, smart-camera consumer devices,
medical imaging consoles, and analogous applications increasingly delegate
scene-interpretation tasks to large multimodal models that ingest entire
frames on every cycle. The four shortcomings recited above apply in equal
measure to such applications, and at substantially greater aggregate cost
given the typically continuous and long-running nature of the source.
A need therefore exists for a vision-delivery primitive that is agnostic
to the source of the frames and that applies the same modulation and
delta-classification techniques uniformly across desktop, surveillance,
robotic, vehicular, medical, and consumer-camera deployments.

A need therefore exists for a system that exposes the host computer's
display to an AI agent in a manner that (i) is deterministic and auditable,
(ii) transmits only that information which has actually changed and which
the agent actually requested, (iii) separates the visual modality from the
textual and structural modalities so each can be queried at its native cost,
and (iv) requires no AI component inside the transmission decision itself.

## 3. Brief Summary of the Invention

The invention divides the frames produced by a visual source into a
deterministic spatial partition of cells, preferably arranged as a regular
grid of twelve columns by eight rows. For each cell, the system exposes to
the external AI agent one or more independent and separately addressable
layers, including without limitation: a visual layer the parameters of
which (resolution, compression quality, color depth, granularity) are
modulable on a per-cell basis; a plain-text layer that extracts textual
content from within the cell without spending vision tokens, where the
source admits such extraction; a layout layer that derives structural
information about user interface elements from a platform accessibility
tree where available; and one or more further optional layers that perform
additional deterministic or machine-learned analyses on a per-cell basis,
such as object detection, face or pose recognition, optical-flow or motion
estimation, semantic segmentation, anomaly detection, or depth estimation.

On each cycle, each cell is subjected to one or more perceptual hash
functions whose output is compared with a previously stored value retained
in an active buffer. Based on this comparison, the cell is classified as
new, changed, or unchanged. Cells classified as new or changed have their
payload transmitted to the agent; cells classified as unchanged are
identified by reference only, and their payload is omitted from the
transmission. The active buffer is then updated with the new state.

The classification operates without any machine-learning component and is
therefore reproducible: identical inputs produce identical classifications.
The active buffer is preserved across cycles, and, upon detection of a reset
trigger such as a change in the displayed origin or a substantial
restructuring of the underlying user interface, the active buffer is
archived into a chronological history, from which prior states may later be
recalled or searched by perceptual similarity.

## 4. Brief Description of the Drawings

The drawings are filed concurrently with this specification.

- **FIG. 1** is a block diagram depicting the overall architecture of a
  system embodying the invention, showing the computational desktop (100),
  the modulated vision layer (200), and an external AI agent (300).
- **FIG. 2** depicts the deterministic grid of twelve columns and eight rows
  superimposed on the desktop viewport (400), together with cells classified
  as new (410), changed (420), and unchanged or cached (430).
- **FIG. 3** depicts the three independent layers — visual (510), plain
  text (520), and layout (530) — exposed per cell (500), together with a
  composed request emitted to the agent (540).
- **FIG. 4** is a flowchart of the per-cell classification pipeline,
  comprising capture (600), perceptual hash computation (610), comparison
  against the active buffer (620), the decision node (630), the three
  branches new (640), changed (650), and unchanged (660), and the buffer
  update (670).
- **FIG. 5** depicts the active buffer (700) indexed by origin and scroll
  bucket, the reset trigger (710), the archive operation (720), and the
  chronological history (730).
- **FIG. 6** depicts the per-cell modulation parameters — resolution (810),
  compression quality (820), color depth (830), and granularity (840) — as
  applied to a source cell (800) to yield a modulated payload (850).

## 5. Detailed Description of the Invention

### 5.1 System architecture

Referring to FIG. 1, an embodiment of the invention comprises a visual
source (100) producing successive frames of image content. In the principal
embodiment described herein, the visual source is the display of a host
computer on which one or more applications (102, 104, 106) and the
operating system user interface (108) are simultaneously rendered. In
alternative embodiments, the visual source is, without limitation: a
webcam or other locally attached image sensor; an Internet-Protocol
network camera; a video-surveillance feed aggregating one or more such
cameras; a robotic-vision pipeline producing frames from one or more
sensors mounted on a robot; a perception camera of an autonomous or
semi-autonomous vehicle; a microscopy, endoscopy, ultrasonography, or
other medical or scientific imaging device; a body-worn or vehicle-
mounted dashboard camera; a satellite, aerial, or unmanned-aerial-vehicle
camera; or any analogous device or service producing a time-varying
sequence of images that an AI agent may need to consume. Interposed
between the visual source (100) and an external artificial-intelligence
agent (300), the modulated vision layer (200) of the invention operates as
a local or network-accessible server.

The modulated vision layer (200) comprises four functional modules: a
capture and grid module (210), which acquires successive frames of the
desktop and partitions each frame into cells according to the grid described
below in connection with FIG. 2; a perceptual hash classifier (220), which
classifies each cell as new, changed, or unchanged according to the pipeline
described below in connection with FIG. 4; an active buffer and history
module (230), which retains previously observed cell states and archives
them when a reset trigger is detected, as described below in connection with
FIG. 5; and a transport module (240), which exposes the foregoing to the
external agent (300) through a standardized protocol, preferably the Model
Context Protocol.

The agent (300) communicates with the modulated vision layer (200) along two
paths: an outbound path on which only those cell payloads that have been
classified as new or changed are transmitted (P2), and an inbound path on
which the agent (300) issues queries or control actions (P3) that the
modulated vision layer (200) may translate into operations upon the desktop
(100) along path P4.

### 5.2 Deterministic grid

Referring to FIG. 2, the desktop viewport (400) is divided into a regular
grid of twelve columns and eight rows, yielding ninety-six cells. Each cell
is identified by a pair of integer coordinates `C(i, j)` where
`i ∈ {0, …, 11}` denotes the column index and `j ∈ {0, …, 7}` denotes the
row index. The grid is deterministic in the sense that, for a given viewport
of dimensions `W × H` pixels, the bounding rectangle of each cell `C(i, j)`
is fully determined by `(i, j, W, H)` and is independent of the content
displayed therein.

The dimensions of the grid — twelve by eight in the preferred embodiment —
may be varied in alternative embodiments. Grids of `M × N` for any positive
integers `M` and `N` are within the scope of the invention, with the
trade-off that finer grids produce more numerous, smaller cells (and
therefore more opportunities to omit unchanged content) at the cost of
greater per-frame computational overhead.

Furthermore, the partition of the viewport into cells need not be
geometrically uniform. In alternative embodiments, the bounding rectangles
of cells may differ from one another in width, height, or both, including
without limitation: partitions in which certain cells are merged or
subdivided to better align with semantic regions of the displayed content;
adaptive partitions that allocate more or smaller cells to regions of
higher visual density or known agent interest; and partitions in which the
cell grid is rotated, sheared, or otherwise affinely transformed relative
to the viewport. The classification, buffer indexing, modulation, history,
and reset-trigger operations described herein apply mutatis mutandis to
such non-uniform partitions. The essential character of the partition is
that it be **deterministic** — meaning that, for a given viewport state and
a given partition strategy, the bounding rectangle of each cell is fully
determined and reproducible — and not that it be geometrically uniform.
The term "grid" as used throughout this specification and the appended
claims is to be construed in this broad sense, encompassing both uniform
and non-uniform deterministic partitions.

### 5.3 Three independent layers per cell

Referring to FIG. 3, for each cell `C(i, j)` (500), the invention exposes
three independent and separately addressable layers to the agent (300).

The **visual layer (510)** encodes the raster content of the cell as a
sequence of bytes. The parameters under which the encoding is performed —
resolution, compression quality, color depth, and granularity — are
modulable independently per cell and per cycle, as described below in
connection with FIG. 6. A single cell may therefore be served at low
resolution and grayscale in one cycle and at full resolution and color in
the next, with no other cell on the screen affected by the change.

The **plain-text layer (520)** extracts textual content from the cell
directly. Where the host operating system provides text-rendering hooks
(such as the GDI text-out interception in Windows, the accessibility text
nodes in macOS, or the equivalent in X11 or Wayland), the text is read from
those hooks. Where it is not so available, an embedded text-extraction
component reads the rendered raster directly. In both cases the agent
receives text strings without paying the vision-token cost of decoding the
underlying raster image.

The **layout layer (530)** derives structural information about user
interface elements from the platform's accessibility tree, including the
type of each control (button, text field, list, menu, and so forth), its
bounding rectangle, and any semantic role it advertises. The agent may
therefore reason about the cell as a structured object — "a button labeled
OK at coordinates (x, y)" — rather than as an opaque image.

The agent (300) may request any subset of the three layers for any subset of
the ninety-six cells on any given cycle. The composed request (540) is
emitted as a single message through the transport module (240). In an
illustrative example, the agent may request the plain-text layer only for
the cells overlapping a known dialog area, the visual layer at full
resolution for a single cell where a graphical decision must be made, and
the layout layer for the entire grid as a structural overview. No other
content is transmitted.

Furthermore, the three layers described above — visual, plain-text, and
layout — are exemplary and not exhaustive. Additional layers may be
incorporated into the modulated vision layer (200) without departing from
the invention, including without limitation: an object-detection layer
that exposes per-cell bounding boxes and class labels produced by a
deterministic or machine-learned detector; a face-recognition or face-
detection layer that exposes per-cell face identities or positions; a
pose-estimation layer that exposes per-cell human or object poses; an
optical-flow or motion-vector layer that exposes per-cell motion
estimates between successive frames; a semantic-segmentation layer that
exposes per-cell pixel-class assignments; an anomaly-detection layer that
flags cells exhibiting statistically unusual content relative to a
learned or recorded baseline; a depth-estimation layer where the source
provides or permits stereo or monocular depth inference; and any other
deterministic or machine-learned per-cell feature extractor that produces
output addressable by the same cell coordinates as the visual layer. Each
such layer is independently requestable, independently modulable, and
participates in the same perceptual-hash classification pipeline
described in section 5.4 below, such that its output is transmitted only
for cells classified as new or changed. The plain-text and layout layers
may be omitted entirely in embodiments where the visual source does not
provide textual rendering or an accessibility tree (as is typically the
case for raw camera feeds), without affecting the operation of the
remaining layers or of the modulation and classification pipeline.

### 5.4 Classification pipeline

Referring to FIG. 4, for each cell on each cycle, the perceptual hash
classifier (220) executes the following pipeline.

At step (600), the cell `C(i, j)` is extracted from the current frame.

At step (610), the classifier computes one or more perceptual hash values
over the cell. In the preferred embodiment, two distinct perceptual hash
functions are combined, yielding `H(C) = (h1, h2)`. Suitable functions
include the difference-hash (`dHash`) and the perceptual-hash (`pHash`)
families known in the art, although other perceptual or content-derived
hashes may be substituted without departing from the invention. The use of
two independent hashes reduces the rate of false equivalences arising from
either hash alone.

At step (620), the classifier retrieves from the active buffer (700) the
previously stored hash value `H'(C)` for the same cell coordinate and the
same scroll bucket, as described in section 5.5 below. The classifier
computes a distance `d = D(H, H')` between the current and prior hashes.
Where the hash is multi-component, `D` is the maximum over the per-component
Hamming distances; in alternative embodiments, `D` may be the sum, the
Euclidean norm, or any other monotonic combination.

At step (630), the classifier evaluates the decision: was a prior hash
`H'(C)` present, and if so, is `d` below or equal to a configurable
threshold `t`? Three branches result.

If `H'(C)` was not present, indicating that no prior observation of this
cell exists, the cell is classified at step (640) as **new**, and its
payload (encoded according to the layer parameters defined in section 5.3)
is transmitted to the agent (300).

If `H'(C)` was present and `d > t`, the cell is classified at step (650) as
**changed**, and its payload is transmitted to the agent (300).

If `H'(C)` was present and `d ≤ t`, the cell is classified at step (660) as
**unchanged**. In this case, the payload is omitted from the transmission;
the agent receives only an identifier referring to the cell, with the
understanding that the previously transmitted content remains valid.

At step (670), the active buffer is updated with `H(C)` and the parameters
under which the cell was encoded, except where the cell was classified as
unchanged, in which case the buffer state is already current and need not be
rewritten.

The pipeline operates without any machine-learning component, and is
reproducible: identical inputs `(C, H'(C), t)` always produce identical
classifications. This determinism is one of the principal advantages of the
invention over alternative approaches that employ probabilistic models to
decide whether the visual content has materially changed.

### 5.5 Active buffer and history

Referring to FIG. 5, the active buffer (700) maintains, for each cell of
the grid, the most recently observed perceptual hash and the parameters of
the most recent encoding. The buffer is indexed by the pair `(origin,
scroll bucket)`, where `origin` is an identifier of the displayed source
(such as a window handle, a uniform resource locator if the source is a web
browser, or any other host-defined identifier of the displayed surface) and
`scroll bucket` is the integer `floor(scroll_y / B)`, where `scroll_y` is
the vertical scroll position of the displayed source and `B` is a
configurable bucket size, preferably one-half the viewport height. The
scroll-bucket index ensures that the same cell coordinate `C(i, j)` at
different scroll positions is treated as a distinct entry, since its
underlying content is in fact different.

Upon detection of a reset trigger (710), the active buffer is archived into
a chronological history (730) at step (720). Reset triggers include, in
order of computational cost: a change in the `origin` identifier; a change
in the title of the displayed source; a transition of the displayed source
into a loading or transitional state; a structural change in the
accessibility tree exceeding a configurable threshold; and a perceptual
change in a majority of cells exceeding a configurable threshold.

The history (730) retains snapshots in chronological order, indexed by
timestamp, origin, and reason for archival. The agent (300) may query the
history by perceptual similarity, supplying a probe hash and a tolerance,
and retrieve any prior snapshot whose hashes fall within that tolerance.
The history is read-only with respect to the agent; it cannot be modified
by query. The history is bounded by a configurable retention policy
(preferably least-recently-used eviction with a maximum number of snapshots
or maximum aggregate size, whichever is reached first).

### 5.6 Per-cell modulation parameters

Referring to FIG. 6, the visual layer of each cell may be encoded under any
combination of the following parameters.

The **resolution parameter (810)** scales the cell's raster content by a
multiplicative factor, preferably one of `{0.25, 0.5, 1.0}`, although any
positive factor may be used. A cell encoded at resolution `0.25` carries
sixteen times less raster data than the same cell encoded at resolution
`1.0`, and may be sufficient for the agent's purposes when only the gross
appearance of the cell is required.

The **compression quality parameter (820)** specifies the lossy compression
ratio applied to the raster, preferably one of `{30, 70, 90}` on a scale of
1 to 100. Lower values reduce data volume at the cost of visual fidelity.

The **color depth parameter (830)** specifies whether the cell is encoded
in one-bit black-and-white, eight-bit grayscale, or full color. Many cells
that contain only text or line drawings convey their information adequately
in grayscale or black-and-white, at substantially reduced cost.

The **granularity parameter (840)** specifies whether the cell is
encoded as edge information only (suitable for rapid layout inspection), as
text-glyph silhouettes (suitable for textual reading), or as full content.
Granularity may be implemented as a pre-processing filter applied prior to
compression.

The modulated payload (850) produced by combining these four parameters is
substantially smaller than the unmodulated raster, while remaining
sufficient for the agent's intended use. The agent may request a coarse
modulation for an initial overview and a finer modulation for cells where
the decision actually lives.

### 5.7 Embodiments and variations

The foregoing description applies primarily to a host computer running a
Windows-family operating system. The invention is not limited to that host.
In alternative embodiments, the host is a computer running macOS, a
distribution of Linux operating system, a mobile operating system such as
Android or iOS, or any other operating system that provides a graphical
user interface and a programmatic mechanism to capture the displayed
content. The accessibility tree referenced in section 5.3 is replaced, in
those alternative embodiments, by the analogous structure of the respective
operating system (NSAccessibility on macOS; AT-SPI on Linux; UIAutomator on
Android; etc.).

The invention is further not limited to computer-display sources. In
alternative embodiments, the visual source (100) is one or more of: a
webcam or other locally attached image sensor; a network camera
communicating over Real-Time Streaming Protocol (RTSP), the Open Network
Video Interface Forum (ONVIF) family of protocols, or any analogous
streaming protocol; a video management system aggregating one or more such
cameras and forming part of a video-surveillance deployment; a robotic-
vision pipeline consuming frames from one or more sensors mounted on a
robot; an autonomous or semi-autonomous vehicle perception stack consuming
frames from one or more cameras on said vehicle; a smart-camera consumer
device such as a doorbell camera, home-security camera, or pet-monitoring
camera; an industrial machine-vision system inspecting items on a
production line; a microscopy, endoscopy, ultrasonography, or other
medical-imaging device; a satellite, aerial, or unmanned-aerial-vehicle
camera; or an analogous device or service producing time-varying image
content. In such alternative embodiments, the plain-text and layout layers
described in section 5.3 may be omitted (when no accessibility tree or
textual overlay exists), reduced in scope, or augmented or replaced by one
or more of the additional layers contemplated at the end of section 5.3.
The modulation, perceptual-hash classification, buffer, history, and
reset-trigger operations described herein apply without modification.

The transport module (240) preferably implements the Model Context Protocol
("MCP") published as an open specification, but is not limited thereto.
Alternative embodiments may employ any protocol that supports a request-
response or request-stream exchange between the modulated vision layer
(200) and the agent (300), including without limitation custom JSON-over-
WebSocket, gRPC, Server-Sent Events, or HTTP/2 streams.

The agent (300) may be any external system that consumes the output of the
modulated vision layer, including without limitation a large language model
operated through a commercial application programming interface, a locally
hosted model executing on the same host as the modulated vision layer, an
automation script that does not employ a model at all, or a human-operated
client used for inspection or debugging.

### 5.8 Implementation notes

A reference implementation of the invention has been constructed in the
Rust programming language, compiled to a native Windows executable, and
distributed under the product name "Verdesk". The reference implementation
has been measured against a baseline agent transmitting full screenshots on
each cycle and has demonstrated a reduction of vision-token consumption of
between approximately 89 and 93 percent on representative interactive
sessions. The measurement harness is included in the implementation as a
reproducible example. Source code of the reference implementation, however,
forms no part of the present specification, the invention being defined by
the claims rather than by any particular implementation.

---

## 6. Claims (informal — preserved for use in the non-provisional)

The provisional application requires no formal claims, but the following
claim set is preserved internally to inform the non-provisional filing
within twelve months hereof.

**1.** A computer-implemented method for delivering content from a visual
source to an external agent, the method comprising:

(a) capturing successive frames from said visual source and dividing each
captured frame into a plurality of cells according to a deterministic
partition, said partition being either a uniform grid or a non-uniform
arrangement of cells;

(b) exposing the captured content to the external agent as one or more
independent and separately addressable layers, said layers including at
least one selected from the group consisting of: a visual layer, a plain-
text layer, a structural layer, an object-detection layer, a face- or
pose-recognition layer, an optical-flow or motion-vector layer, a
semantic-segmentation layer, an anomaly-detection layer, a depth-
estimation layer, and any further deterministic or machine-learned per-
cell feature layer;

(c) applying to each said cell at least one perceptual hash function,
comparing a result thereof against a previously stored value retained in an
active buffer, and classifying each said cell as one of: new, changed, or
unchanged;

(d) transmitting to the external agent only those cells classified as new
or changed, together with identifiers permitting reference to cells
classified as unchanged without retransmitting their content; and

(e) updating said active buffer with the results of said classification.

**2.** The method of claim 1, wherein said visual layer permits, on a
per-cell basis, independent modulation of at least one parameter selected
from the group consisting of: resolution, compression quality, color depth,
and granularity.

**3.** The method of claim 1, wherein said classifying in step (c) is
performed without any machine-learning component and produces reproducible
results given identical inputs.

**4.** The method of claim 1, further comprising: detecting a reset trigger
in respect of said visual source; and, in response to said detection,
archiving said active buffer into a chronological history accessible to
said agent by query of perceptual similarity.

**5.** The method of claim 4, wherein said reset trigger comprises at least
one of: a change in an origin identifier of said visual source, a change
in a title or session identifier of said visual source, a transition of
said visual source into a loading or transitional state, a structural
change in an accessibility tree or analogous structural descriptor
associated with said visual source exceeding a first threshold, or a
perceptual change exceeding a second threshold in a majority of cells of
said partition.

**6.** The method of claim 1, wherein said active buffer is indexed by a
pair comprising an origin identifier of said visual source and a position
bucket derived from a positional parameter of said visual source — said
positional parameter being, by way of example and not limitation, a
vertical scroll position of a displayed document, a pan-tilt-zoom setting
of a camera, a temporal segment identifier of a video stream, or any
analogous parameter of said visual source — such that a single cell
coordinate at distinct values of said positional parameter corresponds to
distinct entries in said active buffer.

**7.** A system comprising one or more processors and a memory containing
instructions which, when executed by said one or more processors, cause the
system to perform the method of any one of claims 1 through 6.

**8.** A non-transitory computer-readable medium storing instructions
which, when executed by one or more processors, cause said one or more
processors to perform the method of any one of claims 1 through 6.

---

## 7. Abstract

A modulated vision layer interposed between a visual source — such as a
computational desktop, webcam, network camera, surveillance feed, robotic-
vision pipeline, autonomous-vehicle camera, medical imaging device, or
analogous time-varying image source — and an external artificial-
intelligence agent divides each frame into a deterministic partition of
cells (preferably a regular grid of twelve columns by eight rows but
optionally non-uniform), exposes each cell through one or more independent
layers (including visual, plain-text, structural, object-detection, face-
or pose-recognition, optical-flow, semantic-segmentation, anomaly-
detection, depth-estimation, and further extensible per-cell feature
layers), applies one or more perceptual hash functions to each cell on
each cycle, and classifies each cell as new, changed, or unchanged by
comparison against an active buffer of prior states. Only cells classified
as new or changed are transmitted; unchanged cells are referenced by
identifier. The classification is deterministic and reproducible. An
active buffer is preserved across cycles and archived to a chronological
history upon detection of a reset trigger. Visual-layer encoding
parameters — resolution, compression quality, color depth, and
granularity — are modulable independently per cell and per cycle. Vision-
token consumption is reduced by approximately 89 to 93 percent on
representative computer-display sessions compared with full-frame agents,
with analogous reductions expected on continuous visual sources of higher
aggregate cost such as surveillance and robotic deployments.
