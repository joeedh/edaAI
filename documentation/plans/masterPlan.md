# edaAI — Master Plan & Program Roadmap

**The top-level program spine. This document defines the project's stages and
serves as the parent index from which all detailed (child) plans are derived.**

Status: Living document. Date: 2026-06-07.

> This is the *governing* plan. It stays high-level on purpose: it defines
> **stages**, their goals, their exit gates, and **which child plans each stage
> will spawn**. Detailed design lives in the per-subsystem reports and in the
> child plans registered in §7 — not here. When the program evolves, update this
> document first, then derive/revise the affected child plans.

---

## 1. Purpose & how to use this document

- **For planning:** each stage below is a unit of program-level work. Before a
  stage begins, its registered **child plans** (§7) are written/refined to
  implementation detail. This document says *what* and *in what order*; child
  plans say *how*.
- **For tracking:** stage **exit gates** (§6) are the milestones. A stage is not
  "done" until its gate criteria pass.
- **For navigation:** §7 is the **plan registry** — the index mapping every
  stage to its source reports and its (existing or future) child plans.

Existing background reports this plan organizes:

| Ref | Title | Role |
|-----|-------|------|
| R-I  | [`initialReport.md`](./initialReport.md) | full library architecture; defines the `eda::*` stack |
| R-II | [`llmAugmentedRouting.md`](./llmAugmentedRouting.md) | inner/outer-loop LLM augmentation model |
| R-III| [`referenceDesignDatabase.md`](./referenceDesignDatabase.md) | reference-design corpus + format conversion feasibility |

---

## 2. Program goal & shape

Build a layered set of C++20 EDA libraries — a verifiable geometry/connectivity/
routing kernel — with a first-class **LLM authoring surface**, so a language
model can author PCBs through a propose → validate → commit loop grounded in a
deterministic engine.

The program is a **downward-only dependency stack** built **bottom-up**: each
stage rests on a stable layer beneath it and delivers something independently
testable. The two non-negotiable cross-cutting commitments — **geometric
robustness** and **determinism** — are established at the bottom and enforced at
every stage above (§5).

```
  Stage 7  LLM-augmented routing & autonomy            (R-II)
  Stage 6  LLM authoring surface  (eda::ai)            (R-I §11)
  Stage 5  Interchange & corpus   (eda::io + DB)       (R-I §15, R-III)
  Stage 4  Manipulation & routing (edit/solve/route)   (R-I §9–10)
  Stage 3  Verification           (eda::drc)           (R-I §8)
  Stage 2  Spatial & model core   (index/doc/edit)     (R-I §6–7,10)
  Stage 1  Geometry kernel        (geom/curve)         (R-I §4–5)
  Stage 0  Program foundations    (build/CI/harness)   (cross-cutting)
```

---

## 3. Stage overview (at a glance)

| Stage | Name | Primary modules | Delivers | Gated by |
|:----:|------|-----------------|----------|----------|
| 0 | Program foundations | build, CI, test/determinism harness | a buildable, tested skeleton + conventions | — |
| 1 | Geometry kernel | `eda::geom`, `eda::curve` | robust coords/predicates, `Shape`, CDT, curves | S0 |
| 2 | Spatial & model core | `eda::index`, `eda::doc`, `eda::edit` | a mutable, indexed board you can transact on | S1 |
| 3 | Verification | `eda::drc` | spatial DRC + connectivity + structured violations | S2 |
| 4 | Manipulation & routing | `eda::edit`, `eda::solve`, `eda::route` | push/pull, octilinear solver, shove/escape/river/diff-pair | S3 |
| 5 | Interchange & corpus | `eda::io`, reference-design DB | read/write real formats; grounding corpus | S2 (io), S3 (verify), R-III legal gate |
| 6 | LLM authoring surface | `eda::ai` | intents/tools/reflection/feedback over the kernel | S3 (min), S4/S5 (full) |
| 7 | LLM-augmented routing | `eda::ai` + `eda::route` | planner/critic loop, heuristic discovery, eval harness | S4, S6 |

Stages 1→4 are strictly sequential (each is the foundation for the next).
Stage 5 can proceed in parallel once Stage 2 (for I/O) and Stage 3 (for verified
ingest) exist. Stages 6–7 ride on the stable kernel.

---

## 4. The stages

Each stage lists its **goal**, **scope**, **key deliverables**, and the
**child plans** it will spawn (registered in §7). Detailed design is deferred to
those child plans and the source reports.

### Stage 0 — Program foundations
- **Goal:** a buildable, continuously-tested skeleton and the conventions every
  later stage depends on.
- **Scope:** CMake project + `litestl` as a submodule; toolchain (C++20/23,
  clang-format from litestl, warnings-as-errors); CI; the **determinism &
  golden-test harness** and **geometry fuzz harness** (used from Stage 1 on);
  WASM build target validated early; legal groundwork for R-III.
- **Deliverables:** repo scaffolding, CI green on an empty lib, contributor docs,
  test harness, decision log.
- **Child plans:** *Build & tooling plan*, *Test/determinism/fuzzing strategy*,
  *Coding standards & API conventions*.

### Stage 1 — Geometry kernel (`eda::geom`, `eda::curve`)
- **Goal:** the trust anchor — robust, deterministic geometry primitives.
- **Scope:** integer-nm coordinates + units; **exact predicates**
  (`orient2d`/`incircle`, segment intersection); the `Shape` variant with the
  uniform bbox/distance/intersect interface; boolean + offset ops; the
  **constrained Delaunay triangulator**; the **arc-length curve API** with line +
  arc backends. (R-I §4–5.)
- **Deliverables:** `eda_geom`, `eda_curve` libraries with fuzz + golden tests.
- **Child plans:** *Geometry kernel plan* (coords/predicates/Shape/boolean),
  *CDT plan*, *Curve API plan*.

### Stage 2 — Spatial & model core (`eda::index`, `eda::doc`, `eda::edit`)
- **Goal:** a board you can represent, index, and mutate transactionally.
- **Scope:** templated shape tree (BVH/loose-KD) + per-polygon polytree BSP,
  incremental/refittable, layer-aware; the board document model (stackup, nets/
  netclasses, padstacks, vias, tracks, zones, stable IDs); the transaction/
  command/undo layer with **dry-run**. (R-I §6–7, §10.)
- **Deliverables:** `eda_index`, `eda_doc`, `eda_edit` (transactions only here;
  push/pull lands in Stage 4).
- **Child plans:** *Spatial index plan*, *Board document model plan*,
  *Transaction/undo plan*.

### Stage 3 — Verification (`eda::drc`)
- **Goal:** the engine's conscience and the LLM's primary feedback channel.
- **Scope:** prioritized rule model + resolution; incremental spatial DRC over
  the index; connectivity/union-find + ratsnest; **typed, structured
  `Violation`** output; ERC hooks. (R-I §8.)
- **Deliverables:** `eda_drc` with incremental re-check + structured results.
- **Child plans:** *DRC rule-model & engine plan*, *Connectivity/ratsnest plan*.

### Stage 4 — Manipulation & routing (`eda::edit`, `eda::solve`, `eda::route`)
- **Goal:** interactive-quality editing and assisted routing.
- **Scope:** push/pull as transactions; octilinear/45° constraint solver +
  length/skew matching; the shove router (walkaround → shove); BGA escape (MMCF)
  + the river/bus first-class object; differential pairs. (R-I §9–10.)
- **Deliverables:** `eda_solve`, `eda_route`, push/pull in `eda_edit`.
- **Child plans:** *Push/pull plan*, *Octilinear solver & length-match plan*,
  *Shove router plan*, *Escape/river/diff-pair plan*.

### Stage 5 — Interchange & corpus (`eda::io`, reference-design DB)
- **Goal:** exchange real designs and assemble the grounding corpus.
- **Scope:** serialization (canonical text format + versioning); IPC-2581
  (canonical archive), KiCad (working), Gerber X2/Excellon/IPC-356 ingest/export;
  netlist import; the tiered converter pipeline and corpus/DB from R-III
  (license-gated, metadata+derived-features, confidence/provenance, PDF best-
  effort). (R-I §15, R-III.)
- **Deliverables:** `eda_io`, converter tools, corpus schema + similarity index.
- **Child plans:** *Serialization & format-versioning plan*, *IPC-2581/KiCad/
  Gerber I/O plan*, *Reference-design corpus & legal plan*, *PDF reconstruction
  pipeline plan*.
- **Note:** the corpus sub-track is **gated by the R-III legal review** (Stage 0
  groundwork) before any third-party files are stored/hosted.

### Stage 6 — LLM authoring surface (`eda::ai`)
- **Goal:** make the kernel legible and drivable by a language model.
- **Scope:** the litestl binding-based reflection surface; three-tier API
  (primitive ops → tools/skills → intents); query/inspection endpoints
  (`whats_near`, `why_unrouted`, `explain_violation`, `ratsnest`); the propose →
  dry-run → validate → commit feedback loop. (R-I §11.)
- **Deliverables:** `eda_ai` with a stable tool/intent surface + structured I/O.
- **Child plans:** *Binding & reflection surface plan*, *Tool/intent API plan*,
  *Query/feedback-loop plan*.

### Stage 7 — LLM-augmented routing & autonomy (`eda::ai` + `eda::route`)
- **Goal:** close the loop — an LLM that plans, critiques, and improves routing
  on top of the deterministic engine, with measured quality.
- **Scope:** intent compiler (spec → constraints); topology/order planner;
  closed critic→repair loop (commit-if-better); offline heuristic discovery;
  RAG over the Stage-5 corpus; the **evaluation harness** (vs. no-LLM baseline).
  (R-II.)
- **Deliverables:** the augmentation services + a benchmark suite + results.
- **Child plans:** *Intent-compiler plan*, *Planner/critic-loop plan*,
  *Heuristic-discovery plan*, *Routing evaluation/benchmark plan*.

---

## 5. Cross-cutting tracks (every stage)

These are not stages; they are commitments enforced *throughout*. Each gets its
own child plan in Stage 0 and is a standing acceptance criterion thereafter.

- **Geometric robustness** — exact predicates + one snap-rounding policy; no
  subsystem invents its own geometry.
- **Determinism & reproducibility** — same inputs ⇒ byte-identical outputs;
  enforced by golden tests in CI; any nondeterminism is a build-breaking bug.
- **Testing & fuzzing** — unit + golden + geometry fuzzing from Stage 1 onward.
- **Performance & incrementality** — index/CDT/DRC update, never full-rebuild;
  the LLM never sits in a geometric hot loop (R-II).
- **WASM-ability** — keep the core hermetic and litestl-WASM-aware; no heavy
  third-party geometry in the core.
- **Provenance & data quality** — for ingested/corpus data and LLM-consumed
  facts: confidence, source, verification status (R-III §8).
- **Legal/licensing** — the R-III gate on third-party design redistribution.

---

## 6. Stage exit gates (milestones)

A stage passes its gate only when these hold (in addition to its child-plan
acceptance criteria):

| Stage | Exit gate | Suggested version |
|:----:|-----------|:-----------------:|
| 0 | CI green; harnesses run; conventions ratified | v0.0 |
| 1 | predicates pass fuzz suite; CDT/curves deterministic & golden-stable | v0.1 |
| 2 | a board round-trips through model + index; transactions undo/dry-run cleanly | v0.2 |
| 3 | DRC + connectivity produce typed violations; incremental re-check verified | v0.3 |
| 4 | a multi-net board routes (assisted) DRC-clean; diff-pair length-matched | v0.5 |
| 5 | IPC-2581/KiCad/Gerber round-trip; open-corpus ingest verified via DRC | v0.6 |
| 6 | an LLM authors a simple board end-to-end via the tool/intent surface | v0.8 |
| 7 | LLM-augmented routing beats the no-LLM baseline on the benchmark suite | v1.0 |

Gates are **decision points**: pass, iterate, or re-scope before committing to
the next stage.

---

## 7. Plan registry (parent → child index)

The authoritative index of plans. **Existing** = already written; **Planned** =
to be authored when its stage is approached. Keep this table current — it is how
future plans are discovered and traced back to this spine.

| Stage | Source report(s) | Child plan | Status |
|:----:|------------------|------------|:------:|
| — | — | `masterPlan.md` (this document) | **Existing** |
| 0 | cross-cutting | Build & tooling plan | Planned |
| 0 | cross-cutting | Test / determinism / fuzzing strategy | Planned |
| 0 | cross-cutting | Coding standards & API conventions | Planned |
| 1 | R-I §4 | Geometry kernel plan (coords/predicates/Shape/boolean) | Planned |
| 1 | R-I §4.5 | Constrained Delaunay triangulation plan | Planned |
| 1 | R-I §5 | Arc-length curve API plan | Planned |
| 2 | R-I §6 | Spatial index plan (shape tree + polytree) | Planned |
| 2 | R-I §7 | Board document model plan | Planned |
| 2 | R-I §10 | Transaction / undo plan | Planned |
| 3 | R-I §8 | DRC rule-model & engine plan | Planned |
| 3 | R-I §8.3 | Connectivity / ratsnest plan | Planned |
| 4 | R-I §10 | Push/pull plan | Planned |
| 4 | R-I §9.2/9.4 | Octilinear solver & length-match plan | Planned |
| 4 | R-I §9.1 | Shove router plan | Planned |
| 4 | R-I §9.3/9.5/9.6 | Escape / river / differential-pair plan | Planned |
| 5 | R-I §11.4 | Serialization & format-versioning plan | Planned |
| 5 | R-I §15, R-III §4 | IPC-2581 / KiCad / Gerber I/O plan | Planned |
| 5 | R-III §7/9 | Reference-design corpus & legal plan | Planned |
| 5 | R-III §6 | PDF reconstruction pipeline plan | Planned |
| 6 | R-I §11 | Binding & reflection surface plan | Planned |
| 6 | R-I §11.1 | Tool / intent API plan | Planned |
| 6 | R-I §11.2/11.3 | Query / feedback-loop plan | Planned |
| 7 | R-II §4.1 | Intent-compiler plan | Planned |
| 7 | R-II §4.2/4.3 | Planner / critic-loop plan | Planned |
| 7 | R-II §4.4 | Heuristic-discovery plan | Planned |
| 7 | R-II §8 | Routing evaluation / benchmark plan | Planned |

---

## 8. Sequencing, parallelism & risk gates

- **Critical path:** S0 → S1 → S2 → S3 → S4. These are sequential; do not start a
  stage before its predecessor's gate passes.
- **Parallelizable:** S5 I/O can begin after S2; the S5 corpus track can begin
  after the S0 legal groundwork and proceed alongside S3/S4. S6's binding surface
  can be prototyped against S3 and grown as S4/S5 land. S7 needs S4 + S6.
- **Riskiest stages:** S1 (geometry robustness — most expensive to get wrong),
  S4 (routing is research-adjacent — ship *assisted*, not autonomous, first),
  S5 PDF track (research-grade; best-effort), S7 (LLM quality must be *measured*,
  not assumed).
- **Re-scope triggers:** if a gate slips badly, prefer narrowing scope (fewer
  backends, assisted-not-autonomous, open-corpus-only) over weakening the
  cross-cutting commitments in §5 — those are load-bearing for trust.

---

## 9. Maintenance of this document

This is a living plan. When direction changes:

1. Update the affected **stage** and its **exit gate** here first.
2. Add/revise the corresponding **child plan** entry in §7 (mark new ones
   *Planned*; link them when *Existing*).
3. Record the rationale in the Stage-0 decision log.

The detailed reasoning behind every stage already exists in R-I/R-II/R-III; this
spine exists to *sequence* that work and to be the single place from which all
future plans descend.
