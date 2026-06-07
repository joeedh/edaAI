# edaAI — Initial Architecture Report

**A set of C++20+ EDA libraries designed for LLM-driven authoring of printed circuit boards.**

Status: Draft / planning. Author-of-record: project owner. Date: 2026-06-07.

> Filename note: the task requested `inititalReport.md`; this file uses the
> corrected spelling `initialReport.md` in the same `documentation/plans/`
> directory. Rename if the literal spelling is required by tooling.

---

## 1. Executive summary

The goal is a layered stack of small, composable, heavily-templated C++20
libraries that together form an *EDA kernel* — the geometry, connectivity,
constraint, and routing engine underneath a PCB CAD tool — with a first-class
**machine-authoring surface** so that a large language model (LLM) can drive
layout the way a human drives a GUI.

The defining design constraint is *not* raw performance or feature breadth; it
is **legibility and controllability by an LLM**. Every subsystem must:

- expose **stable, named, deterministic operations** (an LLM cannot click and
  drag; it issues structured commands);
- produce **structured, machine-parseable feedback** (a DRC violation must be a
  typed object with coordinates and a remedy hint, not a rendered red X);
- be **transactional and reversible** (the model proposes, the engine
  validates, the host commits or rolls back);
- be **introspectable** (the model can query "what is near this pad", "why is
  this net unrouted", "what rule did I violate") cheaply.

This report proposes the module decomposition, the geometry/numeric kernel,
each requested subsystem (spatial tree, polygon tree, constrained Delaunay,
DRC, push/pull, orthogonal-45 solver, BGA escape + river routing, differential
pairs, arc-length curve API), the **LLM-facing layer** that ties them together,
and — importantly — a catalog of subsystems and concerns the original sketch
did *not* mention but which are load-bearing for a real board (Section 12).

The stack builds directly on two existing repositories:

- **`litestl`** — C++20 containers (`litestl::util::Vector/Set/Map/BoolVector`,
  small-buffer-optimized, open-addressed, non-throwing, WASM-aware) plus a
  concepts-based **runtime binding system**. The binding system is the natural
  vehicle for the LLM tool/reflection surface (Section 11).
- **`polytree`** — a BSP-style spatial tree for complex polygons supporting
  point-in-polygon (winding), closest-point-on-edge, line/polygon intersection,
  and polygon containment. This becomes the per-polygon "special tree"
  referenced in the brief.

---

## 2. Design principles

1. **Header-light, dependency-light.** litestl is the only mandatory runtime
   dependency. Geometry robustness code (exact predicates) is vendored, not
   pulled from Boost/CGAL, to keep the build hermetic and WASM-friendly.
2. **One source of truth for geometry.** A single geometry kernel
   (`eda::geom`) defines coordinates, predicates, and the shape variant. No
   subsystem rolls its own point type.
3. **Integer board coordinates.** Board space is integer nanometers (see 4.1).
   Floating point is confined to *derivation* (curve flattening, triangulation
   scratch math) and is always snapped back to the integer grid through robust
   rounding.
4. **Everything is a transaction.** Mutations go through a command/undo layer
   (`eda::edit`). This is what makes push/pull, the constraint solver, and the
   LLM "propose → validate → commit" loop all share one mechanism.
5. **Templated where it pays, type-erased at the boundary.** Spatial trees and
   the curve API are templated for speed and inlining; the LLM-facing and
   serialization boundaries are type-erased/reflected so the surface stays
   small and stable.
6. **Determinism.** Same inputs ⇒ byte-identical outputs. No iteration over
   hash-set order without an explicit sort key; no address-dependent tie-breaks.
   Determinism is what makes LLM behavior reproducible and testable.

---

## 3. Module map

```
                         ┌─────────────────────────────────────────┐
                         │            eda::ai  (LLM surface)        │
                         │  intents · tools · reflect · feedback    │
                         └───────────────▲─────────────────────────┘
                                         │  commands / queries
            ┌────────────────────────────┼───────────────────────────────┐
            │                            │                                │
   ┌────────▼────────┐        ┌──────────▼─────────┐          ┌───────────▼─────────┐
   │ eda::route      │        │ eda::edit          │          │ eda::drc            │
   │ escape · river  │        │ txn · undo · ops   │          │ rules · spatial DRC │
   │ diffpair · shove│        │ push/pull          │          │ connectivity        │
   └────────┬────────┘        └──────────┬─────────┘          └───────────┬─────────┘
            │                            │                                │
   ┌────────▼────────┐        ┌──────────▼─────────┐          ┌───────────▼─────────┐
   │ eda::solve      │        │ eda::doc           │          │ eda::index          │
   │ ortho/45 LP     │◄──────►│ board model        │◄────────►│ spatial tree (BVH/  │
   │ length match    │        │ nets·layers·pads   │          │ KD) · polytree      │
   └────────┬────────┘        └──────────┬─────────┘          └───────────┬─────────┘
            │                            │                                │
   ┌────────▼────────────────────────────▼────────────────────────────────▼─────────┐
   │ eda::geom   coordinate kernel · robust predicates · Shape variant · CDT          │
   │ eda::curve  arc-length curve API (line / arc / … backends)                       │
   └─────────────────────────────────────────────────────────────────────────────────┘
                                         │
                         ┌───────────────▼─────────────────┐
                         │ litestl  util containers · bind  │
                         └─────────────────────────────────┘
```

Proposed namespace / package layout (one library target each, all depend
downward only):

| Namespace      | Library            | Responsibility |
|----------------|--------------------|----------------|
| `eda::geom`    | `eda_geom`         | coordinates, vectors, robust predicates, `Shape` variant, transforms, boolean/offset ops, **CDT** |
| `eda::curve`   | `eda_curve`        | arc-length-parameterized curve concept + backends |
| `eda::index`   | `eda_index`        | templated spatial tree (BVH/loose-KD over shapes) + per-polygon BSP (polytree) |
| `eda::doc`     | `eda_doc`          | board model: layers/stackup, nets/netclasses, pads/vias/footprints, tracks, zones |
| `eda::edit`    | `eda_edit`         | transactions, undo/redo, command objects, **push & pull** |
| `eda::drc`     | `eda_drc`          | rule model, spatial DRC, connectivity/ratsnest, ERC hooks |
| `eda::solve`   | `eda_solve`        | orthogonal/45° constraint solver, length/skew matching |
| `eda::route`   | `eda_route`        | escape routing, river routing, differential pairs, shove-router |
| `eda::ai`      | `eda_ai`           | intent layer, tool registry, reflection, structured feedback |
| `eda::io`      | `eda_io`           | serialization, Gerber/Excellon/IPC-2581/ODB++, netlist import |

---

## 4. Geometry & numeric kernel (`eda::geom`)

This is the foundation; getting it wrong poisons everything above it.

### 4.1 Coordinates

- **Board coordinate = `int64` nanometers** (KiCad uses int nm; we widen to 64
  bit to comfortably hold large panels and intermediate products). A board of
  ±1 m fits in ±10⁹ nm with ~13 decimal digits of headroom in int64.
- `Vec2i` (int64 ×2) is the canonical board point. `Vec2d` (double) exists only
  for scratch math and is never persisted.
- A compile-time `Unit` policy gives ergonomic literals (`5_mm`, `12_mil`,
  `100_um`) that resolve to nm at parse time. LLM-facing APIs accept unit-typed
  values to eliminate the "is this mm or mil?" class of model error.

### 4.2 Robust predicates

Pure integer or naive double geometry produces self-intersecting polygons and
non-manifold routing under shove. We adopt **Shewchuk-style adaptive exact
predicates** (`orient2d`, `incircle`) and exact/snapped segment intersection.
For integer inputs these can often be computed exactly in 128-bit. This is the
single most important reliability investment in the stack; it is what lets the
CDT, boolean ops, and shove router be *provably* non-degenerate.

### 4.3 The `Shape` leaf type

The spatial tree leaves and the renderable/clearance primitives are a closed
variant:

```cpp
namespace eda::geom {
  struct StrokedLine { Vec2i a, b; int64 width; CapStyle cap; };
  struct Rect        { Vec2i min, max; int64 corner_radius = 0; };
  struct Circle      { Vec2i center;  int64 radius; };
  struct Polygon     { util::Vector<Vec2i> outline;
                       util::Vector<util::Vector<Vec2i>> holes; };
  struct Arc         { Vec2i center; int64 radius;
                       Angle start, sweep; int64 width; };   // see eda::curve

  using Shape = std::variant<StrokedLine, Rect, Circle, Polygon, Arc>;
}
```

Every `Shape` must answer a uniform interface used by the index and DRC:
`bbox()`, `distance(Vec2i) -> int64` (signed, negative = inside for closed
shapes), `intersects(const Shape&, int64 clearance) -> bool`, and
`hit(Vec2i, int64 tol) -> bool`. `StrokedLine`/`Arc` are "fattened" segments,
so clearance checks reduce to **segment/segment minimum-distance vs. sum of
half-widths plus clearance** — cheap and exact in integer arithmetic.

`Polygon` is the heavyweight case and gets its own acceleration structure
(4.5 / Section 6).

### 4.4 Boolean & offset operations

Needed for copper pours, thermal reliefs, clearance "shadows", and converting
stroked geometry to filled regions for manufacturing. Provide polygon
clipping (union/intersect/difference/xor) and **polygon offset/inset** (Minkowski
with rounded/mitered/square joins). Implementation: a Vatti/Greiner-style
clipper hardened with the exact predicates, or a vendored, well-tested clipper
behind our `Polygon` type. Offsetting is what generates clearance regions and
teardrops.

### 4.5 Constrained Delaunay triangulation (CDT) — `eda::geom::cdt`

A **constrained Delaunay triangulator** underpins several higher layers:

- **Routing space discretization.** The free space between obstacles is
  triangulated; the dual graph of the triangulation is the search graph for the
  topological router (Section 9). This is exactly how topological autorouters
  (gEDA's Toporouter, academic "rubber-band sketch" routers) work.
- **Zone/pour filling & thermal reliefs.** Triangulating a pour-minus-obstacles
  region yields fillable area and connectivity.
- **Point-location & nearest-feature** queries via the triangulation.

API shape:

```cpp
namespace eda::geom::cdt {
  struct Triangulation {
    // vertices, triangles (indices), half-edge adjacency
    // constraint edges flagged; supports incremental insert/remove
  };
  Triangulation triangulate(span<const Vec2i> points,
                            span<const Segment> constraints,
                            Options);          // returns CDT
  // queries:
  TriId        locate(const Triangulation&, Vec2i);
  // dual graph for routing:
  ChannelGraph channels(const Triangulation&, /*obstacle tags*/);
}
```

Algorithm: incremental insertion with Lawson/edge-flip restoration of the
Delaunay property, then constraint-edge recovery. **Incrementality is a
requirement, not a nice-to-have** — push/pull and routing mutate geometry
constantly, and re-triangulating the whole board per edit is a non-starter.
Use the exact `incircle`/`orient2d` predicates so the result is robust to
collinear/cocircular inputs (extremely common on a grid-snapped PCB).

---

## 5. Arc-length curve API (`eda::curve`)

A generic, backend-pluggable curve abstraction parameterized by **arc length**
(not by an arbitrary `t`). Arc-length parameterization is what makes
length-matching, equal-spacing of vias/teeth, diff-pair gap maintenance, and
"walk N mm along this trace" trivial and exact.

### 5.1 Concept

```cpp
namespace eda::curve {
  template <class C>
  concept Curve = requires(const C c, double s) {
    { c.length() }      -> std::same_as<double>;     // total arc length (nm)
    { c.point_at(s) }   -> std::convertible_to<Vec2d>; // s in [0,length]
    { c.tangent_at(s) } -> std::convertible_to<Vec2d>; // unit tangent
    { c.curvature_at(s) } -> std::convertible_to<double>;
    { c.bbox() }        -> std::convertible_to<Box2i>;
  };
}
```

A `CurveRef` type-erased wrapper (small-buffer, à la litestl) lets the rest of
the system hold heterogeneous curves without templates leaking upward.

### 5.2 Backends

- **`LineSeg`** — straight segment. Trivial closed-form everything.
- **`CircularArc`** — center/radius/start/sweep. Closed-form everything. This is
  the second mandatory backend and the basis for arc-routed (any-angle/rounded)
  traces.
- **`Polyline`** and **`Biarc`** — composites built from the two primitives;
  arc length is the prefix-sum over segments, so `point_at(s)` is a binary
  search over a cumulative-length table + local interpolation.
- **Future:** clothoid/Euler-spiral (curvature-continuous corners, nice for RF),
  cubic Bézier/B-spline (flatten-to-biarc for fabrication). These slot in
  behind the same concept without touching callers.

### 5.3 Fabrication bridge

PCB fabrication ultimately consumes lines and arcs (Gerber G01/G02/G03).
Every curve provides `flatten_to_arcs(tolerance) -> Vector<variant<LineSeg,
CircularArc>>`. Higher-order curves are *authoring* conveniences that always
reduce to the manufacturable primitives, with the tolerance snapped to the
integer grid.

---

## 6. Spatial index (`eda::index`)

Two cooperating structures, matching the brief's "fully templated spatial tree
whose leaves are basic shapes … the latter [polygons] would have their own
special kd tree."

### 6.1 Top-level shape tree

A **fully templated, bounding-volume spatial tree** over `Shape` leaves.
Recommendation: a **loose/bounding-interval hierarchy (BVH) or loose KD-tree**
keyed on integer AABBs, rather than a classic point KD-tree, because PCB
primitives are *extended* (segments, rects) not points.

```cpp
namespace eda::index {
  template <class Leaf,
            class BoundsOf = DefaultBounds<Leaf>,
            int  MaxLeaf   = 8>
  class ShapeTree {
    // build(span<Leaf>), insert(Leaf), remove(handle), refit(handle)
    // query: forEachInBox, forEachWithinClearance(Shape, clr, fn),
    //        nearest(Vec2i), raycast(seg)
  };
}
```

Key requirements driven by the rest of the stack:

- **Incremental & refittable.** Routing/push-pull move thousands of segments;
  the tree supports `refit`/`insert`/`remove` without full rebuild, and a
  deferred-rebuild heuristic when quality degrades.
- **Layer-aware.** Either one tree per copper layer, or a tree key that includes
  a layer mask, so clearance queries are restricted to relevant layers (most
  DRC is intra-layer; pad/via clearance is cross-layer).
- **litestl-backed.** Node/leaf storage uses `util::Vector`; handle→leaf maps
  use `util::Map`. Small-buffer optimization keeps tiny local queries
  allocation-free.

### 6.2 Per-polygon BSP/KD tree (polytree)

Each `Polygon` leaf carries its **own** acceleration structure — the `polytree`
BSP approach: leaves store edge intersections and inside/outside status,
enabling fast **point-in-polygon (winding)**, **closest-point-on-boundary**,
and **segment/polygon intersection**. This is the right tool for pours, complex
keepouts, and footprint courtyards, where a single polygon may have thousands
of edges and naive O(n) tests dominate DRC.

The two-level design (coarse shape tree → per-polygon BSP) means a clearance
query against a pour does `O(log N)` to find the candidate polygon, then
`O(log M)` inside the polygon — instead of `O(N·M)`.

### 6.3 Why this also serves DRC and routing

The index is the shared substrate: DRC iterates leaves and asks the tree for
"everything within clearance"; the router asks for "nearest obstacle to this
ray"; push/pull asks for "what overlaps after I move this." One structure, many
clients — so its query API (Section 6.1) is designed to satisfy all three.

---

## 7. Board document model (`eda::doc`)

The brief is geometry-and-algorithm heavy and *under*-specifies the data model.
A real board needs a connectivity- and stackup-aware model, not just a soup of
shapes. (This is the first of several "things not yet thought of" — see also
Section 12.)

Core entities:

- **Stackup / layers.** Ordered copper + dielectric layers with thickness,
  εr/loss tangent (for impedance), plus technical layers (silkscreen, mask,
  paste, courtyard, fab, keepout). Layer identity is an enum-like `LayerId`.
- **Net / NetClass.** A `Net` is a set of connected pads/segments/vias; a
  `NetClass` carries default trace width, clearance, via type, diff-pair gap,
  impedance target. DRC and routing read rules from netclass with per-net
  overrides.
- **Footprint / Pad / PadStack.** Footprints group pads + courtyard + silk.
  Pads have a padstack (per-layer copper/mask/paste, drill, thermal settings).
- **Track / Via / Arc-track.** Routed copper. A track is a stroked line or arc
  (Section 5) on a layer; a via is a plated through/blind/buried hole spanning a
  layer range.
- **Zone / Pour.** A `Polygon` region with fill rules (thermal relief spokes,
  clearance, min-island removal) that resolves to filled copper polygons.
- **Rule store.** A queryable, prioritized rule set (Section 8).

Everything carries a stable 64-bit `Id` (not a pointer) so commands, undo, the
LLM surface, and serialization can reference objects durably and
deterministically.

---

## 8. Design rule checker (`eda::drc`)

DRC is the engine's conscience and, for an LLM author, its **primary feedback
channel**. It must be incremental and structured.

### 8.1 Rule model

A prioritized, scope-matched rule system (modeled on KiCad's "custom rules" and
Altium's rule classes):

- Rules have a **selector** (net, netclass, layer, area, object type) and a
  **constraint** (min clearance, min/max width, min annular ring, hole size,
  hole-to-hole, edge clearance, courtyard overlap, diff-pair gap/uncoupled
  length, silk-over-pad, etc.).
- Rule resolution is deterministic: highest-priority matching rule wins;
  ties broken by a documented order. The LLM can *query* which rule governs a
  given pair ("why is my clearance 0.2mm here?").

### 8.2 Spatial DRC

For each primitive, query the index (Section 6) for neighbors within
`max_clearance`, then run the exact `Shape`-vs-`Shape` clearance test. This is
`O(N log N)` not `O(N²)`. Checks: clearance, width, annular ring, hole sizes,
hole-to-hole, board-edge, courtyard, zone, silk.

### 8.3 Connectivity / ratsnest

A union-find over pads/tracks/vias/zones per net yields connected components;
unconnected components produce the **ratsnest** (minimum spanning lines of
"still needs routing"). This drives both the LLM's sense of "what's left" and
the router's work list.

### 8.4 Incremental & structured output

- **Incremental.** Edits emit dirty regions; only affected primitives are
  re-checked. The transaction layer (Section 10) reports DRC deltas per commit.
- **Structured.** A violation is:

```cpp
struct Violation {
  RuleId     rule;
  ViolType   type;               // Clearance, Width, Unconnected, ...
  Severity   severity;
  Id         a, b;               // offending objects
  Vec2i      where;              // location for the LLM/user
  int64      measured, required; // e.g. 0.12mm vs 0.20mm
  // optional remediation hint: "increase clearance", "move via", ...
};
```

This typed object is what the LLM consumes to self-correct — vastly better than
a rendered overlay.

---

## 9. Routing subsystem (`eda::route`)

This is the most algorithmically ambitious area and is best built in layers,
each independently useful.

### 9.1 Push & shove interactive router

A KiCad-P&S-style (Tomasz Wlostowski's design) interactive router with
**walkaround** and **shove** modes:

- **Walkaround:** route the new trace around obstacles via the visibility/
  triangulation graph, leaving them fixed.
- **Shove:** treat unfixed traces/vias as movable; pushing the head displaces
  neighbors, propagating collisions outward, each move validated against
  clearance, until the system relaxes or the move is rejected and rolled back.

This sits naturally on three things we already have: the spatial index
(collision queries), the transaction/undo layer (try-move-or-rollback), and the
exact predicates (no degenerate shove geometry). Shove distance and topology
decisions reuse the channel graph from the CDT.

### 9.2 Orthogonal / 45° constraint solver (`eda::solve`)

Manhattan/octilinear routing is a constraint-satisfaction problem: trace
segments are axis-aligned or 45°, must maintain clearance, and should minimize
length/bend count. Approach:

- Represent a route as a sequence of segments with **direction ∈ {0,45,90,…}**
  and free positions.
- Formulate as a **linear program / network-flow** when directions are fixed
  (positions are continuous, clearances are linear inequalities) to find legal
  offsets, with a combinatorial search over the discrete direction/topology
  choices. This is the classic "fix the topology, then solve the geometry"
  split that also underlies length matching.
- Snap-to-octilinear corners use the `CircularArc`/biarc curve backend for
  filleted 45° transitions.

The same LP/flow machinery is reused for **length matching** (9.4) and
**diff-pair gap** maintenance (9.3): these are all "legalize positions subject
to linear constraints" problems.

### 9.3 Differential pairs

A diff pair is two coupled curves with a target **gap** and matched **length**:

- **Coupled routing.** Route a single "center" path through the channel graph,
  then offset it by ±gap/2 using the curve-offset op; corners use arcs to hold
  the gap through bends (where naive 45° corners would violate it).
- **Gap & uncoupled-length rules** are DRC constraints (Section 8) and solver
  constraints (9.2) simultaneously.
- **Skew/length match** within the pair (intra-pair) and across pairs
  (inter-pair) via serpentine/accordion insertion (9.4).

### 9.4 Length / skew matching

Given a target length and a route, insert **tromboning/serpentine** detours.
Because curves are **arc-length parameterized** (Section 5), "add 3.2 mm" is a
direct computation, and the meander is placed where free space exists (found via
the triangulation). Matching proceeds *topology-first* (same via count, same
layer sequence, same general path) then *length-last* — the industry-standard
ordering.

### 9.5 BGA escape (fanout) routing

Escaping a fine-pitch BGA is the hardest local problem. Plan:

- **Dog-bone/fanout generation:** place a via per ball (or shared via per pair)
  in the spoke pattern appropriate to pitch, respecting annular-ring and
  via-in-pad rules.
- **Ordered escape:** the order in which signals leave the field matters — a
  later trace must not be boxed in by an earlier one. Model the perimeter exits
  as a flow problem: **min-cost multi-commodity flow (MMCF)** / ordered-escape
  assignment routes each signal to a boundary slot without crossing conflicts,
  matching the academic and commercial approach to BGA breakout.
- **River formation:** grouped escapes naturally bundle into **rivers** — sets
  of parallel, equal-width, equal-spacing traces that flow together between
  obstacle rows. A river is represented as a *single* parameterized bundle
  (centerline + count + pitch) so the LLM and solver manipulate "the bus", not
  500 individual segments. Rivers expand to individual traces only at commit /
  fabrication time.

### 9.6 River routing as a first-class object

Bundling (a "river"/"bus" abstraction) is the key scalability idea for LLM
authoring: the model reasons about ~dozens of buses, not ~thousands of traces.
The bundle holds: ordered net list, pitch, width, layer, centerline curve, and
fan-in/fan-out maps to pads. The solver keeps members parallel and
length-matched as a group; DRC checks the envelope plus inter-member spacing.

---

## 10. Editing, transactions, push/pull (`eda::edit`)

A command/transaction layer is the backbone that the brief's "push and pull
tool" needs — and that the LLM loop needs even more.

- **Command objects** capture mutations with enough info to undo. Commands are
  data (serializable), enabling replay, scripting, and an LLM **action log**.
- **Transactions** group commands; commit is atomic (all-or-nothing), and a
  failed DRC/legality check rolls the whole transaction back.
- **Push/pull** ("drag a trace/pad and have connected geometry follow") is a
  transaction that: moves the dragged object, propagates to topologically
  connected segments (keeping octilinear constraints via the solver), shoves
  obstacles (9.1), and validates — committing only if legal, otherwise
  presenting the failure to the caller.
- **Preview vs. commit.** Every operation can run in *dry-run* mode returning
  the would-be result + would-be violations without mutating state. This is the
  literal mechanism behind the LLM "propose → validate → commit" loop.

---

## 11. The LLM authoring surface (`eda::ai`)

This is the project's reason for existing and the layer most CAD kernels lack.
It builds on **litestl's concepts-based binding system**, which already exists
to expose C++ to a runtime — here, the runtime is the model.

### 11.1 Three-tier API

1. **Primitive ops** — thin, reflected bindings over `eda::edit` commands
   (`add_track`, `move_via`, `set_netclass`, …). Deterministic, total, typed.
2. **Tools / skills** — composite, validated operations the model calls like
   functions: `route_net(net, hints)`, `escape_bga(ref)`, `match_length(group,
   target)`, `make_diffpair(p, n)`. Each returns a structured result
   (success + objects created, or typed failure + violations).
3. **Intents** — high-level goals the planner decomposes: "route the DDR4 byte
   lane with matched length", "fan out U3 and bus the data lines to the
   connector." Intents produce a *plan* of tool calls that runs transactionally.

### 11.2 Reflection & query surface

The model authors blind unless it can *see*. Provide cheap, structured queries:
`whats_near(point|object, radius)`, `why_unrouted(net)`,
`explain_violation(id)`, `describe_region(box)`, `ratsnest()`,
`rules_for(a,b)`. These read from the index, DRC, and doc model and return
compact JSON-able structures (via the binding system), not pixels.

### 11.3 Feedback loop

```
   intent ──► planner ──► tool calls ──► edit txn (dry-run)
                                            │
                              ┌─────────────┴───────────┐
                              ▼                          ▼
                       DRC violations            success + summary
                              │                          │
                       structured error            commit txn
                              │
                       model revises plan ◄────────────────┘ (loop)
```

Because DRC output is typed (8.4) and ops are reversible (Section 10), the model
can iterate to legality without a human in the loop, and every step is
auditable.

### 11.4 Serialization for humans *and* models

The persisted format should be a **stable, diff-friendly, structured text**
(S-expression like KiCad, or canonical JSON). Two reasons: (a) it round-trips
through git so changes are reviewable; (b) it is directly readable/writable by
the model when it wants to operate on the document as text rather than via the
API. The binding system can generate this surface from the reflected types,
keeping format and model in sync.

---

## 12. Things not yet in the brief (but load-bearing)

The original sketch covers geometry and routing algorithms well but omits
several subsystems without which the libraries cannot author a real,
manufacturable board. Flagging them now prevents painful retrofits.

1. **Connectivity/net model & ERC.** Routing is meaningless without nets;
   electrical-rule checks (unconnected pins, conflicting drivers) complement
   DRC. (Folded into Sections 7–8.)
2. **Layer stackup & controlled impedance.** Diff-pair gap and trace width are
   *consequences* of stackup + target impedance. A field-solver-lite (or table
   lookup) maps (geometry, stackup) → impedance so the LLM can target "90 Ω
   differential" rather than guessing widths.
3. **Padstacks, vias (through/blind/buried/micro), via-in-pad, back-drill.**
   Fanout and HDI routing are impossible without a real via model.
4. **Copper zones/pours with thermal reliefs, anti-pads, island removal.**
   Needs boolean/offset ops (4.4) and per-polygon trees (6.2).
5. **Board outline, cutouts, keepouts, courtyards, mounting/mechanical.**
   Geometry the router and DRC must respect as hard boundaries.
6. **Manufacturing output / DFM.** Gerber X2, Excellon drill, IPC-2581/ODB++,
   IPC-356 netlist, paste/pick-and-place. Plus DFM checks (acid traps, slivers,
   min annular ring, solder-bridge risk). Without export, the board can't be
   built. (`eda::io`.)
7. **Schematic/netlist ingestion.** The board's "intent" (what must connect)
   typically comes from a schematic netlist; provide an import path even if
   schematic capture itself is out of scope.
8. **Teardrops, fillets, mitering.** Reliability/RF geometry generated via
   offset + arc curves; easy to add once 4.4 and Section 5 exist.
9. **Determinism, golden tests, geometry fuzzing.** The exact predicates and
   determinism rules must be backed by fuzz tests (random grid inputs through
   CDT/boolean/shove) and golden-file regression. This is how we trust LLM
   output. **Critical and easy to under-invest in.**
10. **Coordinate robustness at scale.** Snap-rounding policy, how intersections
    that land off-grid are resolved, and how repeated boolean ops avoid drift —
    must be specified once, centrally (4.1–4.2), not per subsystem.
11. **Concurrency model.** The index, DRC, and router are parallelism
    opportunities (per-region, per-net). Decide early: immutable snapshots +
    copy-on-write doc, or single-writer with read replicas. Affects every API.
12. **Versioning / migration of the serialized format.** Boards live for years;
    the format needs a version tag and migration path from day one.
13. **Cost/objective model for routing.** "Good" routing is multi-objective
    (length, vias, bends, congestion, impedance). A shared, tunable cost
    function lets the solver, router, and LLM planner agree on what "better"
    means.
14. **Unit & type safety at the API boundary** (4.1) to stop the LLM confusing
    mil/mm/nm — a surprisingly high-value, low-cost guardrail.
15. **Observability for the LLM:** stable object IDs, an action/undo log it can
    read back, and "explain" endpoints (11.2). Treat the model as a user who
    needs an inspector, not just an API.

---

## 13. Suggested build-out order (phased roadmap)

Each phase produces something testable and independently useful.

| Phase | Deliverable | Unlocks |
|-------|-------------|---------|
| 0 | `eda::geom` kernel: coords, units, **robust predicates**, `Shape`, bbox/distance/intersect | everything; the trust anchor |
| 1 | `eda::curve` line+arc backends; `eda::index` shape tree + polytree BSP | clearance queries, rendering, picking |
| 2 | `eda::geom::cdt` (incremental CDT) + boolean/offset ops | zones, routing graph, teardrops |
| 3 | `eda::doc` model (layers/nets/pads/vias/tracks/zones) + `eda::edit` transactions/undo | a real board you can mutate |
| 4 | `eda::drc`: spatial DRC + connectivity/ratsnest + structured violations | the feedback channel |
| 5 | `eda::edit` push/pull + `eda::route` shove (walkaround first, then shove) | interactive-quality editing |
| 6 | `eda::solve` ortho/45 + length matching; diff pairs | constrained, matched routing |
| 7 | `eda::route` BGA escape (MMCF) + river/bus abstraction | high-density authoring at bus granularity |
| 8 | `eda::ai`: bindings, tools, intents, query/feedback surface | LLM-driven authoring end-to-end |
| 9 | `eda::io`: serialization + Gerber/Excellon/IPC-2581 export | manufacturable output |

Phases 0–4 are the "hard to change later" foundation and deserve the most
rigor (predicates, determinism, the doc model, DRC structure). Phases 5–9 are
features built on a stable base.

---

## 14. Key risks & mitigations

- **Geometric robustness** — *the* classic EDA failure mode. Mitigate with exact
  predicates (4.2), a single snap-rounding policy, and aggressive fuzzing (12.9).
- **Incremental everything** — CDT, index, and DRC must update, not rebuild;
  designing the dirty-region/refit APIs up front (Sections 6/8) avoids a rewrite.
- **Router scope creep** — autorouting is a research field. Ship the *interactive*
  shove router + escape/river/diff-pair *assists* first; full autonomous routing
  is explicitly later/optional.
- **LLM surface churn** — keep the model-facing API small and stable (three
  tiers, 11.1); let it grow by composition (tools), not by widening primitives.
- **Determinism regressions** — enforce with golden tests in CI; any nondeterminism
  is a build-breaking bug, because it breaks LLM reproducibility.

---

## 15. Dependencies & conventions

- **C++20** (concepts, `std::variant`, ranges, `constexpr` math). C++23 where
  available for `std::expected`-style error returns (matches litestl's
  non-throwing style).
- **litestl** as a submodule: `util` containers everywhere, `binding` for the
  `eda::ai` reflection surface, plus its `math` and `io` utilities.
- **polytree** algorithm (BSP polygon tree) vendored/ported into `eda::index`
  as the per-polygon structure.
- **No heavy third-party geometry** (no Boost.Geometry/CGAL) in the core, to
  stay hermetic and WASM-targetable (litestl is already WASM-aware). Vendored,
  self-contained exact-predicate and clipper code only.
- **Error handling:** non-throwing, value-returning (`expected`-like), matching
  litestl conventions; the LLM surface converts these to structured results.
- **Style:** follow litestl's `.clang-format`; nested namespaces `eda::<mod>`.

---

## 16. Summary

The plan is a downward-only stack: a robust integer geometry kernel with exact
predicates and a CDT at the bottom; a templated shape index with per-polygon
BSP trees and an arc-length curve API as the spatial/parametric layer; a
transactional board document with DRC and connectivity as the model; push/pull,
an octilinear constraint solver, and a shove router with BGA-escape/river/
diff-pair support as the manipulation layer; and — the differentiator — an
LLM-facing intent/tool/feedback surface built on litestl's binding system on
top.

The two ideas most worth emphasizing, because they are what make this *for LLMs*
rather than just another router kernel, are: (1) **everything is a reversible,
dry-runnable transaction with typed, structured DRC feedback**, giving the model
a closed propose-validate-commit loop; and (2) **buses/rivers (and curves) are
first-class objects**, so the model reasons about dozens of intents instead of
thousands of segments. The geometry-robustness and determinism investments in
Phases 0–4 are what make all of it trustworthy.

---

### Sources

- KiCad interactive push-and-shove router (Tomasz Wlostowski): kicad-developers
  mailing-list archives —
  https://archive.lists.launchpad.net/kicad-developers/msg13351.html ,
  http://amichalec.net/2013/10/06/push-and-shove-in-kicad/
- BGA escape / ordered escape (MMCF) & river routing —
  https://resources.pcb.cadence.com/blog/2019-best-pcb-routing-methods-for-bga-escape-routing
- Topological autorouting (Situs) —
  https://resources.altium.com/p/automated-pcb-routing-with-situs-topological-autorouter
- Differential-pair placement & topology-first length matching —
  https://autocuro.com/blog/differential-signal-placement-and-routing ,
  https://resources.pcb.cadence.com/blog/pcb-routing-essentials-for-the-modern-designer
- litestl (containers + binding system) — https://github.com/joeedh/litestl
- polytree (BSP polygon spatial tree) — https://github.com/joeedh/polytree
