# Defensive Publication — Verdesk

**Date of public disclosure: 2026-05-28**

**Author / Inventor: Camilo Stefan Brossard Olave**

**Domicile: Santiago de Chile, Chile**

**Public repository (this file): https://github.com/chamilonster/verdesk-skill**

---

## Purpose of this document

This document constitutes a **defensive publication** under standard patent law principles. By publicly disclosing the technical design of Verdesk on the date stated above, the author establishes **prior art** with verifiable timestamp.

Effect of this publication:

1. **The invention described in [`TECHNICAL-SPEC.md`](./TECHNICAL-SPEC.md) becomes part of the public state of the art as of 2026-05-28.**
2. **No third party may obtain a valid patent on the same invention with a filing date later than 2026-05-28.** Any subsequent patent application by a third party covering the same subject matter is subject to rejection by USPTO, EPO, CNIPA, INAPI, INPI Brasil, JPO, KIPO, or any other patent office on grounds of anticipation (lack of novelty).
3. **The author preserves the right to use, develop, distribute, sell, license, and otherwise commercially exploit the invention without obstruction from later-filed third-party patents covering the same subject matter.**

## What this is NOT

- This is **not** an exclusive license. The author does not grant or restrict third-party use of the technology beyond what applicable copyright and trademark law independently provide.
- This is **not** a patent. The author retains the right to file patent applications on this invention in any jurisdiction within applicable grace periods (notably, the 12-month grace period under 35 U.S.C. 102(b)(1) for filings in the United States).
- This document does not waive copyright in the source code or other proprietary materials. The Verdesk software, including its source code, brand, and product implementation, remains the property of the author and is distributed under the terms of the separate licenses applicable to those materials (see `LICENSE` and the End User License Agreement at https://verdesk.app).

## Technical scope of the disclosure

The full technical specification of the invention being publicly disclosed is contained in [`TECHNICAL-SPEC.md`](./TECHNICAL-SPEC.md), filed concurrently with this notice.

In summary, the invention concerns a **modulated visual pipeline interposed between a visual source (such as a computational desktop, webcam, network camera, surveillance feed, robotic vision system, autonomous vehicle camera, medical imaging device, or analogous time-varying image source) and an external artificial-intelligence agent**, characterized by:

1. Division of each frame of the visual source into a deterministic partition of cells (preferably a regular grid of twelve columns by eight rows, optionally non-uniform).
2. Exposure of each cell through one or more independent and separately addressable layers, including without limitation visual, plain-text, structural, object-detection, face-or-pose-recognition, optical-flow, semantic-segmentation, anomaly-detection, depth-estimation, and further deterministic or machine-learned per-cell feature layers.
3. Application of one or more perceptual hash functions to each cell on each cycle, with deterministic classification of each cell as new, changed, or unchanged by comparison against an active buffer of prior states.
4. Transmission to the external agent of only those cells classified as new or changed, with cells classified as unchanged referenced by identifier rather than retransmitted in full.
5. Preservation of the active buffer across cycles and archiving to a chronological history upon detection of a reset trigger; the history is queryable by perceptual similarity to a probe hash.
6. Per-cell modulation of visual-layer encoding parameters, including resolution, compression quality, color depth, and granularity, independently per cell and per cycle.

The complete technical description, including alternative embodiments, claims structure, and the six accompanying figures, is contained in [`TECHNICAL-SPEC.md`](./TECHNICAL-SPEC.md).

## Verifiability of the publication date

The publication date of this document is verifiable via:

- The git commit timestamp of the addition of this file to the public repository at https://github.com/chamilonster/verdesk-skill, which is cryptographically signed and immutable.
- Any subsequent capture of this repository by the Internet Archive (https://web.archive.org/).
- Public announcements made by the author concurrent with this publication on social media platforms, including but not limited to https://twitter.com/chamilonster.

## Contact

For inquiries, contact Camilo Stefan Brossard Olave at camilo.brossard@gmail.com.
