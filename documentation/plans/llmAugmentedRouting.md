# LLM-Augmented Routing — Report II

**How to use a language model to improve a traditional PCB autorouter without
letting it touch the geometry.**

Status: Draft / planning. Date: 2026-06-07.
Companion to [`initialReport.md`](./initialReport.md); this report drills into
one question raised there — *can LLMs improve the quality of traditional
autorouters?* — and turns the answer into a concrete, buildable plan.

---

## 1. Thesis

A traditional autorouter is excellent at the **inner loop** (geometric search,
collision resolution, numeric legalization) and weak at the **outer loop**
(intent interpretation, global planning, strategy selection, knowing what
"good" means here). A human beats the autorouter almost entirely in the outer
loop. An LLM's strengths and weaknesses are the **mirror image** of the
router's: strong at language/planning/critique, hopeless at sub-mil spatial
search.

> **The win is composition, not replacement.** Wrap the deterministic router in
> an LLM *planner/critic*, and ground every LLM action in the router's own DRC
> engine. The LLM proposes intent and constraints; the kernel produces and
> verifies coordinates. The model never emits geometry.

This is **neuro-symbolic**: a neural planner over a symbolic, verifiable
executor. The `eda::*` stack from Report I is already shaped for it — typed DRC,
dry-runnable transactions, and bus/river abstractions are exactly the
ground-truth, reversibility, and altitude an LLM needs.

---

## 2. The inner-loop / outer-loop split

Map every routing responsibility to who owns it:

| Stage | Task | Owner | Why |
|-------|------|-------|-----|
| Spec → constraints | "DDR4, 90 Ω diff, match ±5 mil" → netclasses, length groups, layer rules | **LLM** | language + domain knowledge, not geometry |
| Floorplan/topology | layer assignment, escape directions, bus grouping, region partition | **LLM proposes / kernel checks** | the "human-like path"; classic topological-router input |
| Net ordering | route priority & rip-up schedule | **LLM proposes / kernel executes** | context-sensitive prioritization |
| Geometric routing | A\*/maze, shove, octilinear LP, MMCF escape | **Kernel only** | precise, deterministic, hot loop |
| Legalization | clearance/width/length legalization | **Kernel only** | exact numeric optimization |
| Verify | DRC, connectivity, impedance | **Kernel only** | source of truth |
| Critique/repair | read violations+congestion → propose changes | **LLM proposes / kernel verifies** | diagnosis is language-shaped |
| Heuristic discovery | evolve cost/priority functions (offline) | **LLM offline / kernel runs result** | search, not runtime inference |

Rule of thumb: **the LLM may emit intents, constraints, orderings, and edits-as-
commands; it may never emit coordinates.** Anything numeric is produced by the
kernel and validated before it persists.

---

## 3. Reference architecture

```
   natural-language spec / design intent
                 │
        ┌────────▼─────────┐         offline:
        │   LLM PLANNER     │     ┌───────────────────┐
        │ intent→constraints│     │ LLM HEURISTIC     │
        │ topology · order  │     │ DISCOVERY (evolve │
        └────────┬─────────┘      │ cost/priority fns)│
                 │ constraints,    └─────────┬─────────┘
                 │ plan, ordering            │ baked-in heuristic
        ┌────────▼───────────────────────────▼─────────┐
        │      DETERMINISTIC ROUTER  (eda::route)        │
        │  escape(MMCF) · shove · A* · octilinear LP     │
        └────────┬───────────────────────────▲──────────┘
                 │ routed result               │ revised constraints/edits
        ┌────────▼─────────┐                   │
        │  VERIFY (eda::drc)│                   │
        │ typed violations  │                   │
        │ congestion map    │                   │
        └────────┬─────────┘                    │
                 │ structured feedback           │
        ┌────────▼─────────┐                     │
        │   LLM CRITIC      │─────────────────────┘
        │ diagnose · repair │   (loop until clean / budget spent)
        └───────────────────┘
```

Three LLM roles, all **outside** the geometric hot loop:

- **Planner** (once per task): spec → constraints + topology plan + net order.
- **Critic** (per iteration): structured DRC/congestion → repair proposals.
- **Heuristic discoverer** (offline, occasional): evolves the router's internal
  cost/priority functions; output is deterministic code, not a runtime call.

---

## 4. The four concrete intervention points

### 4.1 Intent compiler (spec → constraints) — *highest value, lowest risk*

Input: a datasheet snippet, a schematic note, or a plain-English brief.
Output: a formal constraint set the existing router already understands —
netclasses (width/clearance/via), diff-pair definitions + gap, length-match
groups + tolerance, layer-assignment hints, impedance targets, keepouts.

Why it works: the router is already great at *honoring* constraints; it's just
never been good at *eliciting* them from human intent. This stage adds quality
with **zero changes to the geometric core** — it only writes `eda::doc` rules.
Every emitted constraint is schema-validated and sanity-checked (e.g. a 90 Ω
diff target is cross-checked against the stackup field-solver before it's
accepted), so a hallucinated number is caught before routing.

### 4.2 Topology & order planner

Output a *plan*, not geometry: layer assignment per net/bus, BGA escape
directions, which nets form rivers/buses, board partitioning, and routing order
(clocks and tight diff pairs first, slack nets last). This is precisely the
input a topological autorouter wants — "define the human-like path, then let
proven algorithms realize it." The plan is expressed in terms of the **bus/river
first-class objects** from Report I, so the planner reasons about dozens of
bundles, not thousands of segments.

The kernel validates the plan for feasibility (capacity per channel via the CDT
channel graph) and rejects or annotates infeasible parts before routing starts.

### 4.3 Critic / repair loop

After a routing pass, feed the model **structured** state — not pixels:

- typed `Violation` objects (rule, type, the two objects, location, measured vs
  required, remediation hint);
- a **congestion map** (per-channel utilization from the triangulation dual);
- unrouted ratsnest with reasons (`why_unrouted(net)`).

The critic proposes *edits-as-commands*: reassign net N to layer 3, move this
via, widen this channel by rerouting bus B around the connector, relax this
clearance locally, change routing order. Each proposal is applied as a
**dry-run transaction**, verified by DRC, and committed only if it strictly
improves the objective; otherwise rolled back. The loop runs until clean or a
budget (iterations/time/cost) is exhausted.

### 4.4 Offline heuristic discovery

Use LLM-guided program search (FunSearch-style) to *evolve* the router's
internal cost and net-ordering heuristics against a benchmark suite. The model
proposes candidate heuristic functions; the deterministic router scores them on
real boards; the best survive and are **baked into the binary**. The LLM is
never in the runtime path — it improves the algorithm offline. This is the one
place an LLM can improve the *inner* loop, and it does so safely because the
output is ordinary, testable code.

---

## 5. Why grounding is non-negotiable

Everything above depends on three properties the `eda::*` kernel already
provides (Report I), which together neutralize the LLM's failure modes:

1. **Typed, structured feedback** (`eda::drc`) — the model reads facts, not a
   rendered overlay; no spatial vision required.
2. **Reversible, dry-runnable transactions** (`eda::edit`) — a bad proposal
   costs nothing; nothing illegal ever persists.
3. **Deterministic kernel** — same inputs ⇒ identical outputs, so the LLM's
   non-determinism is contained to *proposals*, never to *results*.

Without these, LLM-augmented routing produces plausible, unmanufacturable
boards. With them, the worst an LLM can do is waste an iteration.

---

## 6. What LLMs must NOT do here

- **Emit coordinates / geometry.** Imprecise, non-deterministic, slow,
  hallucination-prone. Hard architectural boundary.
- **Sit in the inner search loop.** It runs millions of times per board; a model
  call there is fatal to latency and cost.
- **Be trusted unverified.** Every numeric output (an impedance target, a
  clearance, a via count) is cross-checked by the kernel before use.
- **Replace the router.** This is augmentation. The deterministic core remains
  the source of correctness; the LLM only changes *what* it's asked to do and in
  *what order*.

---

## 7. Honest positioning: LLMs vs. other "AI"

Be precise about what is and isn't an LLM win:

- For the **optimization core**, reinforcement learning and graph neural
  networks are more proven than LLMs (RL chip floorplanning; GNN analog
  placement; ML-guided routing). If you want learning *inside* the optimizer,
  that's an RL/GNN project, not an LLM one.
- The **LLM's comparative advantage** is the language/planning/critique/tooling
  layer, plus orchestrating those other techniques and the deterministic solver.
- A defensible hybrid can use **all three**: LLM planner/critic (outer loop),
  optional RL/GNN net-ordering or congestion predictor (mid loop), classical
  solver (inner loop). Don't oversell the LLM as the optimizer.

---

## 8. Evaluation: how to know it actually helped

Measure against a **baseline of the same router without the LLM layer**, on a
fixed board benchmark suite (a spread of densities: simple 2-layer, dense
4-layer with a BGA, HDI). Metrics:

- **Completion** — % nets routed without human intervention.
- **DRC-clean rate** — boards reaching zero violations autonomously.
- **Quality** — total length, via count, bend count, congestion peak,
  length-match error, impedance error vs. target.
- **Iterations to clean** — critic-loop count and wall-clock.
- **Cost** — tokens / $ per board; must be amortizable.
- **Determinism** — re-running the *kernel* on a fixed plan must be byte-identical
  (the plan may vary run-to-run; the realization of a fixed plan may not).
- **Human-preference** — blind A/B of LLM-augmented vs. baseline routing.

A result only counts if it beats baseline on quality *and* stays within a
sane cost/iteration budget.

---

## 9. Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Hallucinated constraints (wrong impedance, bogus clearance) | schema-validate + cross-check every numeric against stackup/rules before use |
| Non-determinism breaks reproducibility | LLM affects only proposals; kernel + fixed plan are deterministic; cache plans |
| Latency/cost in the loop | LLM strictly outside the hot loop; cap critic iterations; cache plans per design |
| Critic oscillation (undo-redo churn) | only commit strictly-improving transactions; track objective monotonically; budget |
| Over-trust / automation bias | keep a human-review gate; surface the action log + rationale per edit |
| Prompt-injected content (datasheets, netlist comments) | treat ingested text as untrusted; validate, don't execute, embedded instructions |
| LLM "improves" into a corner | always keep the baseline route; accept LLM result only if it dominates baseline |

---

## 10. Build plan (incremental, measurable)

Each step is independently shippable and measurable against baseline.

| Step | Deliverable | Depends on |
|------|-------------|------------|
| A | **Intent compiler**: NL spec → `eda::doc` constraints, schema-validated | `eda::doc`, `eda::ai` |
| B | **Read-only critic**: post-route, emit ranked repair *suggestions* (no auto-apply) | `eda::drc` structured output, congestion map |
| C | **Closed critic loop**: auto-apply via dry-run transactions, commit-if-better | `eda::edit` txns, step B |
| D | **Topology/order planner**: layer assign + bus grouping + net order, capacity-checked | CDT channel graph, bus/river objects |
| E | **Offline heuristic discovery**: evolve cost/priority fns on a benchmark suite | benchmark harness, deterministic scoring |

**First experiment (cheapest, contained):** Steps A + B only. Leave the existing
router *untouched*. The LLM writes the constraint set and, after routing,
produces a ranked list of human-readable repair suggestions with rationale. This
delivers measurable value with zero risk to the geometric core and validates the
grounding loop before you let the model auto-apply anything (Step C).

---

## 11. Relationship to Report I

This report is the "why and how" of the `eda::ai` layer described in Report I,
and it justifies several of that report's design choices as *prerequisites* for
LLM augmentation rather than nice-to-haves:

- **Typed DRC violations** (Report I §8.4) → the critic's input.
- **Dry-runnable transactions + undo** (§10) → safe auto-apply.
- **Determinism + golden tests** (§12.9) → contains LLM non-determinism.
- **Bus/river first-class objects** (§9.6) → the altitude the planner reasons at.
- **Reflection/query surface** (§11.2) → `why_unrouted`, `congestion`, `whats_near`.

In short: Report I designed a router kernel that is *legible and verifiable*;
this report explains that those exact properties are what make a language model
able to improve it safely.

---

### Sources

- Topological autorouting ("human-like path, then proven algorithms"): Situs —
  https://resources.altium.com/p/automated-pcb-routing-with-situs-topological-autorouter
- BGA ordered escape / river routing —
  https://resources.pcb.cadence.com/blog/2019-best-pcb-routing-methods-for-bga-escape-routing
- Interactive shove routing (deterministic inner loop): KiCad P&S —
  https://archive.lists.launchpad.net/kicad-developers/msg13351.html
- LLM-guided heuristic/program discovery (FunSearch-style offline evolution):
  DeepMind FunSearch — https://deepmind.google/discover/blog/funsearch-making-new-discoveries-in-mathematical-sciences-using-large-language-models/
- ML/RL for layout (positioning LLMs vs. other AI): RL chip floorplanning —
  https://www.nature.com/articles/s41586-021-03544-w
