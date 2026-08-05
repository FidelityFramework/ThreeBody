# ThreeBody: Clef + Composer Feature Gap-List

> What must be expressed in the Clef language, the Composer compiler, and the
> runtime/platform to fully realize the visual demonstration. Status flags:
> **built** · **partial** (scaffolding or twin exists) · **designed** (specified,
> not coded) · **unspecified** (design call still open). Companion to
> [`gustafson-synthesis-design.md`](gustafson-synthesis-design.md).

## Critical path

**The Tier-3 representation-seal *syntax* (Clef, Milestone 1, currently
OPEN/unbuilt) is the one gap that gates a credible demo.** The un-softened `1/r²`
force kernel has no nonzero lower bound in dataflow, so Tier-1 intrinsic selection
cannot bound it and falls through — correctly. The developer seals b-posit32+quire
at the force site instead. This lets the no-softening force kernel be sealed
*without* the unbuilt research-grade real interval domain, so the keynote has a
real compiler to point at. Everything else can be scaffolded or measured; without
the seal there is no design-time honesty story.

## Build order

1. **CRITICAL — Tier-3 representation-seal syntax** (Clef, M1). The one
   buildable-for-keynote path that does not require the real interval domain.
2. **The ThreeBody integrator itself** (runtime): symplectic leapfrog +
   independent high-precision reference + `G = 1` normalization +
   swap-only-the-number-type across the whole force+integration path. `src/` is
   empty; the curve is the demo's actual instrument.
3. **Linear-momentum drift sensor** + the `[<Invariant>]` declaration for
   energy / angular / linear momentum. Cheapest high-value change; currently
   omitted.
4. **Four-pane residual harness including the FP64+Kahan control.** Load-bearing
   — without it the posit edge is unfalsifiable. Locate the surviving edge in
   `qᵢ−qⱼ` and `1/r²`.
5. **Milestone 0 + M1 resolver**: `RepresentationSelection` coeffect, `Real`
   sentinel, `selectRepresentation`, `R_cov` + ULP objective, capability
   predicates — so the four panes are four bindings of one source, not four
   hand-written programs.
6. **Milestone 3 quire nanopass** (fixed 800-bit, eS=5, precision-independent, no `k`-cap) — the
   single capability the headline most load-bears on.
7. **Structural reversibility contract** (`SymplecticStep` adjoint channel) + the
   typechecked badge on both panes.
8. **THEN, as a clearly-labeled third pane and open research direction**: the
   runtime ubox method (Gustafson's proposal) + its Composer-certified
   conservation-manifold projection operator. Adds a demonstration; does not gate
   the honest two-pane read.
9. **LAST / do not block on**: Milestone 2 real interval domain (research-grade,
   dominant cost) and Milestone 4 Tier-2 `Fidelity.Physics` (spec-blocked).

## Clef language features

| Status | Feature | Why | Gustafson |
|---|---|---|---|
| designed | `[<Invariant>]` conserved-quantity declaration on sim state | Names a `State -> float<dim>` as conserved so Composer can verify it computes without phantom Clifford grades (Tier 1/2) and instrument it at runtime for the drift readout. Serves both the diagnostic and the constraint job. | #2: 18-D state, 7 conserved quantities |
| designed | Linear-momentum drift sensor, expressible + instrumented | Zero from rest → any nonzero value is pure rounding; the tightest sensor and cleanest falsifiable claim. Flagged OMITTED in `Numeric_Selection` §13. | #2 (convergent with the corpus's arithmetic-side sensor) |
| designed | Representation-agnostic reversibility contract: `SymplecticStep = { forward: State -> State; adjoint: Neg<State> -> Neg<State> }` | The structural half of the demo. `Neg` rides the adjoint channel and certifies the inverse is structurally complete; it does not constitute the reversal (live `Φ⁻¹ = S∘Φ∘S`). Type-checks identically on IEEE and posit. **Hard constraint:** do not type the velocity flip as `Neg<Velocity>` (category error). | Refines #6's latent "reversibility ⇒ conservation" |
| designed | Tier-3 representation-seal form at an expression site | **Critical path.** Seals the force kernel to b-posit32+quire without the unbuilt real interval domain. Syntax is OPEN. | Enables #3 (no softening): keeps true `GMm/(r₁−r₂)²` without the compiler pretending it is bounded |
| designed | Minimum-approach / non-zero-lower-bound annotation on `r` | The honest substitute for softening's `+h`: bound the *domain* of validity, do not deform the force law. Must come from a range law / companion attribute, never a hard-coded constant. | #3/#4: bound the domain instead of softening the singularity |
| partial | Conservation laws as Tier-2 constraint predicates (`[<Requires>]`/`[<Ensures>]`, QF_LIA/QF_BV via Z3) | Certifies invariants are *computed* correctly under the chosen representation (structural zeros stay zero via the quire). Design-time guarantee, distinct from runtime ubox filtering. | #5: the 7 laws as filter constraints, at compile time |
| designed | Quire / exact-accumulation as a selectable mode with **exactness** semantics | The mechanism that survives the FP64+Kahan control at `qᵢ−qⱼ` and `1/r²`. No "tapered precision near zero / golden zone" framing anywhere. Corrected config: eS=5, fixed 800-bit (independent of precision); delete "39% faster decode" (cite Jonnalagadda et al.'s measured decoder wins, arXiv:2603.01615). | #6: posit advantage by experiment; the quire is the specific mechanism |

## Composer compiler features

| Status | Feature | Why | Gustafson |
|---|---|---|---|
| partial | `RepresentationSelection` coeffect + `Real`/`FloatWidth 0` sentinel + `selectRepresentation` resolver + capability predicates (M0 + M1 core) | Slots a real-number flow into the existing integer selection frame. Traversal skeleton, coeffect carriage, sentinel/resolver pattern, diagnostic plumbing reused verbatim from the integer twin. | Lets the four panes be four target-bindings of one source |
| designed | Real interval domain: outward-rounded FP arithmetic, sign-crossing `recip: RealInterval -> RealInterval list` (0–2 unbounded pieces), transcendentals, terminating widening (M2) | The dominant unbuilt cost and biggest research risk — a **new** abstract interpreter, not a port of `IntervalAnalysis.fs`. For un-softened `1/r²` the sound behavior is to return unbounded pieces and make Tier-1 fall through **loudly**. The demo must not wait on it. | #5 (ubox enclosure) and #4 (trajectory-over-step) both need interval arithmetic; this is the compile-time enclosure, distinct from the runtime ubox |
| partial | FPGA0001-analogue unobservable-range diagnostic firing on the un-softened force | When neither a Tier-2 min-approach law nor a Tier-3 seal is present, Composer must emit a hard, honest diagnostic and never silently soften (no implicit `+h`). The loud-failure contract is what makes no-softening defensible. | #3: refusing `+h` means the compiler must say "range unobservable" rather than fabricate a floor |
| designed | Quire nanopass: recognition pattern, fixed 800-bit sizing (precision-independent, a vector of 25 32-bit integers), per-product-fit QF_BV obligation, per-target capability lowering, `quire-capability-failure` (M3) | The capability the demo most load-bears on. Quire-exact accumulation of the pairwise-force dot products is what survives the Kahan control. Gate Native on FPGA/Xposit, fail loudly on a target lacking it (e.g. Loihi 2). No `k`-cap on sizing. | #6: the quire is the accumulation mechanism behind the posit edge |
| partial | Codata-carriage of representation choice through MiddleEnd lowering, re-checked per pass | Representation choice is codata on the PSG, navigated not recomputed; certified passes preserve by construction, uncertified passes get a per-edge QF_BV Z3 re-check. The conservation invariant **would travel** as a Tier-≤3 fact through Composer's MiddleEnd in MLIR's SMT dialect as the computation lowers, re-checked per pass — so the FPGA pipeline **is designed to** run the same conservation-respecting step the source described. Bounds the **lowering** integrity, distinct from any claim about temporal error growth. | Correction 2: same *shape* as the ubox; Composer is designed to emit and certify the runtime ubox, not execute it |
| designed | Decidability gate on symbolic interval evaluation (total, terminating sub-language; terminating widening) | Symbolic interval evaluation restricted to a total, terminating sub-language. The terminating-widening operator is the well-founded measure bounding the **analysis** — categorically distinct from any claim about temporal error growth of the dynamics. | #1: the rank-function/terminating-widening connection is analysis-side; must **not** be conflated with the dynamics-side linear-bound claim |
| designed | Tier-2 `Fidelity.Physics` integration surface + symbolic interval-evaluator over the restricted sub-language (M4) | Lets `open Fidelity.Physics.OrbitalMechanics` supply a range law bounding `r` away from zero so the force kernel selects via Tier 2. Spec-blocked: ratify Design A vs B first. Type-algebra-only, so the bound is a range law, not a constant. | #3/#4: the principled alternative to softening |

## Runtime / platform features

| Status | Feature | Why |
|---|---|---|
| designed | Symplectic time-reversible integrator (leapfrog/Verlet) + independent high-precision reference | The demo's instrument is a reversal-residual-vs-N curve; the reference keeps secular method-drift from masquerading as arithmetic signal. Normalize `G = 1`. `src/` is empty — must be built. |
| designed | Four-pane (then five-pane) residual harness | IEEE FP64, b-posit32+quire, fixed-point bit-exact, FP64+Kahan control, plus the ubox third pane. The Kahan control is load-bearing. Bookkeeping rule: do not measure drift on a path you constrained by projection. |
| designed | Runtime 18-D ULP-box enclosure + conservation filter + re-projection (the ubox method) | Gustafson's proposal as a runtime mechanism inside whatever representation Composer selected. A numerical-method choice in the running program, **not** a Composer pass. The published method carries the surviving **set** forward; the centroid collapse is informal email shorthand and must be flagged as a heuristic. |
| designed | Dimensionally-sound conservation-manifold projection operator | The re-projection the ubox applies. The PSG **is designed to** emit and certify it (prove it dimensionally sound at Tier 1/2, lower the invariant checks faithfully) — it does not execute it. The runtime deliverable whose lowering-integrity the compile-time enclosure guarantees. |
| partial | Per-target quire/posit capability: Native on FPGA/Xposit, loud failure elsewhere | `PlatformContext.RuntimeModel` carries b-posit HW / Xposit / quire support; a target lacking the quire triggers a capability-coeffect failure, never a lossy fallback. The close-encounter regime is where emulated-vs-native actually shows. |

## What already exists to lean on

- The complete **integer** representation-selection pipeline that numeric
  selection is the real-number twin of: `IntervalAnalysis.fs`, the `IntWidth 0`
  sentinel, the `narrowType` resolver, the `FPGA0001` hard error, check-time
  diagnostics, platform capability facts. Traversal skeleton, coeffect carriage,
  sentinel/resolver pattern, and diagnostic plumbing reuse verbatim.
- The **PSG codata-carriage discipline**: representation choice rides as codata
  beside dimension/grade/escape-class, transported by the fixed-point combinator,
  navigated not recomputed, certified passes preserving by construction,
  uncertified passes getting a per-edge QF_BV Z3 re-check.
- The **three-readings-of-one-traversal** frame: width inference (spatial),
  depth/budget inference (temporal), numeric selection (the third reading).
- **Fixed-point** is already a selectable representation — the bit-exact limiting
  pane comes for free.
- The **four-tier verification model** and the demonstrated-vs-architectural /
  loud-failure governing principles the no-softening honesty story rests on.
- `Numeric_Selection_Implementation.md` §13 already mandates the demo's correct
  shape (`G=1`, swap-only-the-number-type, symplectic + reference, the
  linear-momentum sensor, the Kahan control, fixed 800-bit/eS=5, reversibility as
  horizon extension). The shape is written down; the code is not.

## Research-grade risks (do not block the demo on these)

- **The real interval domain (M2)** — a new abstract interpreter, the single
  biggest research risk. The demo cannot wait on it; build the no-softening force
  on a Tier-3 seal instead.
- **Gustafson's "linear bounds" claim (#1) is false for trajectory enclosure** —
  positive largest Lyapunov forces exponential divergence (Nepomuceno), and naive
  axis-aligned boxes add the wrapping-effect blow-up. Rescuable *only* by changing
  what is bounded to the conservation manifold, where slow growth is an **open
  conjecture**. Kahan is the standing counter-voice. If mentioned, present as open
  conjecture; bound the invariant, never the trajectory.
- **A guaranteed linear-growth bound that travels through compilation is not
  buildable on current infrastructure** — would require the unbuilt interval
  domain plus a formalization surviving the decidability gate. The
  terminating-widening rank function bounds the *analysis*, not the dynamics.
- **The runtime ubox method and its projection operator are entirely unbuilt and
  corpus-absent** (Gustafson's external proposal). The centroid step is informal
  shorthand, not a documented primitive.
- **Tier-2 `Fidelity.Physics` (M4) is spec-blocked and library-unbuilt.**
- **Conserved-quantity count honesty** — full problem has 10 algebraic integrals;
  "7" is correct only in the barycentric frame; the current 2D sim collapses it to
  4. Decide 2D-with-4 or 3D-with-7 before the talk.
- **Posit-over-IEEE advantage is experiment-dependent and unproven** (Gustafson's
  own #6); without the Kahan control pane it is unfalsifiable. Present as a
  measured result, never a foregone conclusion.
