# Reference-Design Database & Format Conversion — Report III

**Feasibility of building a database of manufacturers' reference / template PCB
designs, with tooling to convert sources (including PDF-only ones) into a common
format.**

Status: Draft / feasibility study. Date: 2026-06-07.
Companion to [`initialReport.md`](./initialReport.md) and
[`llmAugmentedRouting.md`](./llmAugmentedRouting.md). Motivation: a curated
corpus of known-good layouts is the natural **retrieval/grounding substrate**
for the LLM-driven authoring stack — both as templates to instantiate and as
examples to learn house conventions from.

---

## 1. Verdict up front

| Dimension | Feasibility | Note |
|-----------|-------------|------|
| Catalog/index of reference designs | **High** | mostly data engineering |
| Ingest of *already-structured* sources (Gerber/IPC-2581/ODB++/KiCad) | **High** | formats are open or readable |
| Ingest of proprietary native CAD (Allegro/Altium/Zuken binary) | **Medium** | partial via importers/exports; coverage gaps |
| **PDF → geometry** | **Medium** | vector PDFs yes; raster needs CV, lossy |
| **PDF → full board (netlist, drill, components)** | **Low–Medium** | research-grade reconstruction; best-effort + human review |
| "**All** manufacturers" as literal completeness | **Not feasible** | unbounded; scope to a prioritized, growing corpus |
| **Redistribution** of the designs themselves | **Low / blocked** | the real constraint — see §7 |

**Bottom line:** The technical pipeline is feasible in tiers, and a high-value
*metadata + derived-feature* index is clearly buildable. The dominant risk is
**not** engineering — it is **licensing**. Most vendor reference designs are
explicitly *evaluation-only, no-redistribution*. The project's design must be
shaped around that fact from day one (§7), or it is dead on arrival regardless
of how good the converters are. The PDF-only tail is the hardest *technical*
piece and is realistically a best-effort, human-reviewed enrichment — not a
fully automatic, trustworthy reconstruction.

---

## 2. Motivation & how it plugs into the stack

For the `eda::*` stack, a reference-design corpus is the retrieval layer that
makes LLM authoring grounded rather than hallucinated:

- **Templates** — "give me a known-good 4-layer buck-converter layout for this
  controller" → instantiate and adapt, rather than route from scratch.
- **Convention learning** — fanout patterns, decoupling placement, diff-pair
  geometry, layer-stack choices distilled from real boards.
- **RAG for the planner/critic** (Report II) — retrieve similar boards to seed
  topology plans and sanity-check the model's proposals.

This reframes the goal: we do not strictly need pixel-perfect, redistributable
*copies* of every board. We need a **searchable index of structured features**
(stackup, net topology classes, component classes, geometry statistics,
embeddings) with links back to the authoritative source. That reframing is also
what makes the legal problem tractable (§7).

---

## 3. What "reference / template designs" are, and where they live

The target corpus, roughly in order of value:

- **Chip-vendor evaluation boards / reference designs** — TI (Reference Design
  Library, TIDA/PMP), Analog Devices (CN circuits, EVAL boards), ST, NXP,
  Microchip, Infineon, Nordic, Espressif, Renesas. Usually ship schematic PDF +
  BOM, frequently Gerbers + native CAD (often Altium), sometimes only PDF.
- **Module/board vendors & open hardware** — Adafruit, SparkFun, Arduino,
  Raspberry Pi, Olimex, many on GitHub in **KiCad/Eagle** with open licenses.
- **EDA tool demo/reference projects** — KiCad demos, Altium examples.
- **Application-note layouts** — often *PDF-only* figures inside app notes; the
  hard tail.

Formats encountered, by richness:

| Format | Geometry | Netlist | Stackup | Components/BOM | Open? |
|--------|:--------:|:-------:|:-------:|:--------------:|:-----:|
| IPC-2581 | ✓ | ✓ | ✓ | ✓ | **open (IPC)** |
| ODB++ | ✓ | ✓ | ✓ | ✓ | Siemens-controlled |
| KiCad `.kicad_pcb` | ✓ | ✓ | ✓ | ✓ | **open** |
| Native CAD (Altium/Allegro/Zuken) | ✓ | ✓ | ✓ | ✓ | proprietary/binary |
| Gerber X2/X3 + Excellon + IPC-356 | ✓ | ✓* | partial | partial* | **open** |
| Gerber RS-274X (plain) | ✓ | ✗ | ✗ | ✗ | open but lossy |
| **PDF (vector)** | partial | ✗ | ✗ | OCR-only | n/a |
| **PDF (raster/scan)** | CV-only | ✗ | ✗ | OCR-only | n/a |

\* Gerber X2 adds netlist/attribute info; IPC-356 carries a testable netlist;
together they recover much of what plain Gerber loses, *if present*.

---

## 4. Choice of common (canonical) format

Two distinct needs: a **rich canonical archive format** and a **working/editable
format**.

- **Canonical archive: IPC-2581.** It is the only *open*, vendor-neutral,
  single-file (XML) standard that carries copper image, **layer stackup**,
  **netlist** (for bare-board/ICT), and **component BOM** together. It is
  exportable from every major EDA tool and is gaining adoption (especially
  aerospace/defense/automotive), without licensing fees. ODB++ is richer in
  practice and more widely supported by fabs, but it is Siemens-controlled — fine
  as an *ingest* source, wrong as our *canonical* target.
- **Working/editable format: KiCad `.kicad_pcb`.** Open, fully documented,
  git-diffable, with a real data model and a growing toolchain (and it is the
  format our own `eda::doc` model maps onto most directly). Use it whenever a
  design must be opened, edited, or rendered.
- **Fallback ingest: Gerber X2/X3 + Excellon + IPC-356.** The lingua franca for
  the long tail; lossy but ubiquitous.

Strategy: **ingest anything, normalize to IPC-2581 as the archive, project to
KiCad for editing/rendering, and store derived features in the database
(§6).** Our internal `eda::doc` model is the in-memory hub all three pass
through.

---

## 5. Source tiers by tractability (the conversion pipeline)

Process the corpus in tiers; harvest the easy, high-fidelity 80% first and treat
the PDF-only tail as best-effort enrichment.

### Tier 0 — already structured & open (HIGH fidelity)
IPC-2581 / ODB++ / KiCad / Eagle / Gerber-X2-with-IPC-356. Parse → `eda::doc` →
normalize to IPC-2581 + derived features. Mostly a parsing + schema-mapping job.
This is the backbone of the database.

### Tier 1 — proprietary native CAD (MEDIUM)
Altium/Allegro/OrCAD/Zuken binary. Routes:
- Prefer the vendor's **own exported** Gerber/ODB++/IPC-2581 when shipped
  alongside (common for eval boards) — sidesteps the binary entirely.
- Use existing importers (KiCad's Eagle/Altium import; commercial translators;
  CAM tools) where available.
- Accept coverage gaps for closed binaries with no public reader (e.g. Allegro
  `.brd`); these fall back to whatever interchange/Gerber the vendor published.

### Tier 2 — plain Gerber + drill (MEDIUM, lossy)
Copper geometry imports cleanly (KiCad's Gerber→PCB import does the bulk
automatically). The hard parts are **reconstruction**, not import:
- **Trace centerlines.** Gerber stores *filled* copper, not centerlines;
  recovering traces = medial-axis/skeleton extraction with width inference.
- **Netlist.** Requires the **drill (Excellon) file** to know layer-to-layer
  connectivity — *without drill data you cannot reconstruct nets.* With it,
  trace-following + via-stitching yields nets (what CAM350 / PCB Tracer do).
- **Components.** Plain Gerber has none; inferred only from silk + pad clusters,
  or recovered from IPC-356/X2 if present.

### Tier 3 — PDF (LOW–MEDIUM, the frontier; see §6)
The genuinely hard case and the reason this report exists.

---

## 6. The PDF conversion pipeline (the hard part)

A PDF is a *rendering*, not a design. Converting it back to a board is an
**inverse problem** — partial, probabilistic, and best done with human review.
Honest decomposition:

### 6.1 Triage
Classify the PDF: **vector** (path/fill/stroke operators present) vs.
**raster/scanned** (image XObjects only). Vector is tractable; raster is CV and
much lower fidelity. Also detect *what* the PDF is — layout plot, dimensioned
mechanical drawing, schematic, or assembly drawing — each needs a different
extractor.

### 6.2 Vector geometry extraction
Use a PDF engine (MuPDF/pdfium/pdfminer) to pull path operators, fills, strokes,
and colors. Then:
- **Layer separation** by color/line-style + nearby legend text (OCR the layer
  key: "Top Copper", "L2 GND", "Silk"). This mapping is heuristic and a frequent
  failure point.
- **Vectorize/snap** geometry to the integer grid of our `eda::geom` kernel;
  fattened strokes → `StrokedLine`/`Arc`, fills → `Polygon`.
- **Infer drills** from concentric pad/annulus pairs; **infer outline** from the
  board-edge layer.

### 6.3 Raster path (scanned PDFs)
Image segmentation + classical CV (or an ML detector) to find copper regions,
pads, silk, ref-des text. This is adjacent to the PCB reverse-engineering and
schematic-digitization literature; expect *coarse* geometry, not
manufacturing-grade. Strictly best-effort, always human-reviewed.

### 6.4 Connectivity & component reconstruction (hardest)
- **Nets:** trace-following + via stitching across separated layers → net graph.
  Confidence drops sharply when layers overlap visually or drill data is absent.
- **Components:** silkscreen OCR for ref-des + outline matching to a footprint
  library; values/BOM from an accompanying schematic PDF (symbol detection +
  wire tracing + label OCR — itself an active ML research problem).
- **Schematic↔layout cross-link:** if both PDFs exist, match ref-des to recover
  intent (net names, part values) the layout alone can't supply.

### 6.5 Output with provenance & confidence
Emit `eda::doc` → IPC-2581/KiCad, but **every reconstructed entity carries a
confidence score and provenance** (which PDF, which page, which heuristic).
Low-confidence designs go to a **human review queue**; they are never silently
treated as ground truth — feeding an LLM auto-reconstructed garbage is worse
than omitting the design (§8).

> Realistic expectation: vector-PDF **geometry** is recoverable to useful
> fidelity; full **electrical** reconstruction (clean netlist + verified BOM)
> from PDF alone is partial and human-supervised. Scope accordingly.

---

## 7. Legal & licensing — the actual blocker

This determines whether the project is viable at all and must lead the design,
not be bolted on.

**The terms are hostile to redistribution.** Surveying the major vendors:

- **TI** — limited, non-transferable, non-sublicensable license; *"shall not
  grant rights … to any third party."* Internal evaluation use only.
- **ST** — title and all IP in the board design *remain with ST*; users may not
  copy in whole or in part or remove notices.
- **ADI** — narrow license tied to use *with ADI devices*; redistribution only
  under terms no less restrictive.

So **mirroring and re-hosting the design files is, in general, not permitted.**
Aggregating them into a public database would breach these EULAs and infringe
copyright. This is the single biggest finding of this report.

**Design patterns that stay legal:**

1. **Metadata + derived-features index, not a file mirror.** Store
   *descriptions, statistics, embeddings, topology classes, links* — not the
   copyrighted files. Link out to the authoritative vendor page for the actual
   download. (Mirrors how parts search engines operate.)
2. **License-gated tiers.** Separate the corpus by license: **open/permissive**
   (OSHW, CERN-OHL, MIT/CC, KiCad demos, much of GitHub hardware) may be stored
   and redistributed; **evaluation-only** vendor designs are *indexed by
   metadata only*, fetched on demand by the end user under their own acceptance
   of the vendor's terms.
3. **Local/private ingestion.** Provide the *converters as tools* so a user can
   ingest designs **they** are licensed to use into **their own** private
   database. The shippable product is the *pipeline*, not necessarily a public
   data dump.
4. **Abstracted learning.** For LLM training/retrieval, prefer derived,
   non-reproducing representations (statistics, embeddings, anonymized
   topology) over verbatim copies — reduces both legal and "regurgitation" risk.
5. **Clean-room / opt-in.** Seek explicit redistribution permission or
   partnerships with vendors for a curated set; default to opt-in, not opt-out.

**Recommendation:** Build the **open-licensed corpus** as the redistributable
core, and treat proprietary reference designs as a **metadata-only index +
user-side local ingestion** capability. This preserves nearly all the value for
LLM grounding while staying on the right side of the EULAs. Get qualified legal
review before any public hosting of third-party design files.

---

## 8. Data-quality risks specific to LLM use

Because this corpus is meant to *ground* an LLM, bad data is actively harmful:

- **Garbage-in amplification.** A mis-reconstructed netlist taught as "known
  good" produces confidently wrong layouts. → Confidence-gate everything;
  exclude low-confidence reconstructions from the training/retrieval set.
- **Provenance is mandatory.** Every fact must trace to a source + extraction
  method so the planner/critic (Report II) can weight it and a human can audit.
- **Verification pass.** Run ingested designs through our own `eda::drc` and
  connectivity checks; flag designs that don't pass as "geometry-only,
  electrically unverified."
- **Dedup & versioning.** The same eval board appears in many formats/revisions;
  canonicalize by (vendor, board id, revision) to avoid double-counting.

---

## 9. Database design

Each entry (canonical record), regardless of source tier:

- **Identity:** vendor, board/part id, family, revision, source URL(s).
- **License:** SPDX-style tag + redistributable flag (drives §7 tiering).
- **Structured design:** stackup, layer count, board dimensions, net topology
  summary, component classes/BOM, design-rule profile.
- **Geometry:** normalized IPC-2581 blob (only if license permits storage);
  otherwise a derived-feature vector + thumbnail.
- **Embeddings:** for similarity search ("buck converter, 2 MHz, 4-layer, QFN")
  — the RAG hook for Reports I/II.
- **Provenance & confidence:** source format, extraction method, per-entity
  confidence, DRC/connectivity verification status.

Stores: a relational/document store for records + a vector index for similarity.
The converters write `eda::doc`; the DB stores the normalized output + features.

---

## 10. Phased plan

| Phase | Deliverable | Risk |
|-------|-------------|------|
| 0 | **Legal review** + license taxonomy; define redistributable vs. index-only tiers | — |
| 1 | **Tier-0 ingest** (IPC-2581/ODB++/KiCad/Eagle/Gerber-X2) → `eda::doc` → normalized archive + features | low |
| 2 | **Database + similarity search** over the open-licensed corpus; RAG hook | low |
| 3 | **Tier-1/2 converters** (native-CAD via vendor exports; Gerber+drill reconstruction with confidence) | medium |
| 4 | **Metadata-only index** of proprietary reference designs + user-side local-ingest tool | legal-gated |
| 5 | **PDF vector pipeline** (geometry + heuristic layers + OCR) with human-review queue | medium |
| 6 | **PDF raster + schematic digitization** (CV/ML, best-effort) | high/research |

Phase 0 gates everything. Phases 1–2 deliver real value from the **open** corpus
with minimal legal/technical risk and should ship first. The PDF work (5–6) is
the research frontier and should be scoped as best-effort enrichment, never the
foundation.

---

## 11. Relationship to Reports I & II

- The converters all target the **`eda::doc`** model and `eda::geom` kernel from
  Report I — this database is a *client* of the same core, not a separate stack.
- Ingested designs are verified through the same **`eda::drc`** engine, so
  "electrically verified" is a first-class, trustworthy flag.
- The corpus is the **retrieval/grounding substrate** the planner and critic of
  Report II assume: templates to instantiate, examples to learn conventions,
  similar-board lookup to seed and sanity-check LLM proposals.
- Derived-feature + provenance discipline mirrors Report II's insistence that the
  LLM consume **structured, attributable** facts, not raw artifacts.

---

## 12. Summary

Building the *catalog and conversion tooling* is feasible and high-value;
building a *redistributable mirror of every vendor's design files* is not —
because of licensing, not engineering. The realistic, defensible shape is: a
redistributable core from **open-licensed** hardware, a **metadata + derived-
feature index** (with user-side local ingestion) for proprietary reference
designs, and a tiered converter pipeline that takes the easy structured 80%
to high fidelity while treating **PDF reconstruction as best-effort, confidence-
scored, human-reviewed enrichment.** IPC-2581 is the canonical archive format,
KiCad the working format, and the whole thing feeds the LLM stack as a grounded
retrieval substrate — provided data quality and provenance are treated as
first-class, because for an LLM, a wrong "known-good" example is worse than none.

---

### Sources

- IPC-2581 vs ODB++ (open single-file standard; netlist+stackup+BOM; adoption) —
  https://resources.altium.com/p/pcb-production-file-format-wars ,
  https://pcbsync.com/ipc-2581/ ,
  https://www.zuken.com/us/blog/ipc-2581-the-open-road-to-reducing-pcb-design-workload/
- Gerber→KiCad reconstruction & limits (no components; drill required for nets) —
  https://forum.kicad.info/t/reverse-engineering-kicad-project-from-gerber-files/30903 ,
  https://www.kicad.org/discover/gerber-viewer/ ,
  https://pcbtracer.com/
- Reverse-engineering / netlist extraction tooling —
  https://pcbsync.com/how-to-convert-gerberodb-back-to-pcb-design-file-reverse-engineering/
- Reference-design sources — https://www.ti.com/reference-designs/index.html ,
  https://www.analog.com/en/resources/evaluation-hardware-and-software/evaluation-boards-kits.html
- Licensing / redistribution terms (the blocker) — TI:
  https://www.ti.com/lit/ml/sszo061/sszo061.pdf ; ST:
  https://www.st.com/resource/en/evaluation_board_terms_of_use/evaluationproductlicenseagreement.pdf ;
  ADI:
  https://www.analog.com/en/design-center/evaluation-hardware-and-software/adi_evaluation_board_license_agreement.html ,
  https://www.analog.com/en/support/software-license-agreement.html
