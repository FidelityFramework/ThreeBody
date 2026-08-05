# Computational Integrity as a Carried, Certified Invariant

> **Status:** design intuition, developed — not a shipped capability or a proven
> theorem. The interval domain that would operationalize the compile-time half is
> research-grade and unbuilt (`Numeric_Selection_Implementation.md` §3.2); the
> runtime linear-bound is Gustafson's open conjecture. Architectural register
> throughout. Higher-altitude companion to
> [`gustafson-synthesis-design.md`](gustafson-synthesis-design.md) and
> [`gustafson-synthesis-gaplist.md`](gustafson-synthesis-gaplist.md).

## The thesis

Most computing systems optimize a single scalar — precision, throughput, or
training loss — and let every structural law of the computation degrade as an
uncounted side effect: IEEE rounding erodes a conservation quantity, the wrapping
effect inflates an enclosure, gradient accumulation leaks energy into Clifford
grades the algebra forbids. This framework treats the structural law of the
computation as a **carried, certified invariant, co-equal with the optimization
target**, from source through every lowering pass toward the heterogeneous
substrate, where it is designed to be re-checked rather than re-derived from the
final binary.

The method is Gustafson's ubox — blur a state into ULP-wide tiles and keep only
the tiles that satisfy the governing law. The mechanism is the PSG-integrity
principle — a declared invariant rides the program-structure graph as a Tier-1-to-3
fact re-checked at each MiddleEnd pass. The applications are three domains where
the law *is* the point of the computation, not a nicety. The three-body reversal
is the proof-of-life: the law is "total linear momentum is exactly zero from
rest," and a system that preserved it through forward integration and live
reversal comes home, with any nonzero residual reading out as pure accumulated
arithmetic error and nothing else.

## ubox as a PSG integrity principle

An enclosure of a computation, carried alongside that computation and re-validated
at every transformation that could perturb it, is the unit of integrity in
Gustafson's ubox method and in our Program Semantic Graph alike. Three moves
recur in both:

1. **Blur** the exact object to a sound enclosing set.
2. **Constrain** that set by an invariant the object must obey, discarding the
   part of the enclosure the invariant forbids.
3. **Re-canonicalize** the survivor into the representation the next stage
   consumes, without losing soundness.

We theorize that ubox is one instance of this shape and that our
PSG-through-MiddleEnd lowering discipline is another. The two instances must be
held sharply apart by **which object is enclosed and when**:

- **The compile-time enclosure** is the static `RealInterval` coeffect the PSG
  carries per node (range selection, quire sizing, bound proofs): a decidable,
  outward-rounded superset of a node's value, deferred to target-binding and
  re-checked per pass. Its integrity question is integrity of the *lowering* —
  does the representation selected for this node still soundly cover its range
  after this pass refined the dialect?
- **The runtime ubox** is the dynamic enclosure of the actual evolving 18-D
  physical state, conservation-filtered at simulation time. Its integrity
  question is integrity of the *trajectory* — does this state still conserve
  momentum after this step?

These are the **same shape over different objects.** The PSG does not *execute*
the runtime ubox. Under this design it is intended to *emit and certify* it: prove
the conservation-projection operator dimensionally sound (Tier 1), make the
per-step enclosure tight via the quire's exact accumulation, and carry the
invariant checks through lowering so the emitted FPGA b-posit pipeline **is
designed to** run the same conservation-respecting step the source described.

### The mapping, slot by slot

| ubox move | PSG analogue | Why the analogy is tight — or loose |
|---|---|---|
| **ULP-box** — blur each rounded state value to `[x−ULP, x+ULP]`, tessellate the next-step region into 1-ULP tiles (a runtime method) | **`RealInterval` coeffect** — an outward-rounded `{Lo; Hi; Dim}` superset of a PSG node's value, a sibling of width inference, carried as a coeffect deferred to target-binding. Outward rounding is the soundness discipline, exactly as the ULP pad is | **Not the same box.** `RealInterval` encloses the *set of possible values a node could take across inputs*, statically, to pick a representation; the ULP-box encloses *one concrete evolving state* at sim time. Integrity of lowering vs integrity of trajectory. The corpus domain *soundly refuses* the un-softened `1/r²` (sign-crossing reciprocal returns unbounded pieces) — the right compile-time answer, and precisely the regime the runtime ubox conservation-filters instead. Cleanest evidence the two boxes are different objects |
| **Conservation-filter** — eliminate every tile that violates the conserved quantities; the surviving set is the part consistent with the invariant | **A typed invariant re-checked per pass** — a developer-declared `[<Invariant>]` on `State -> float<dim>` becomes a Tier-≤3 fact traveling through the MiddleEnd in MLIR's SMT dialect, re-discharged at each lowering edge by the consequence rule | The runtime filter prunes *physical states* during execution. The compile-time re-check prunes *compiler passes* that would silently drop the carried invariant. Both are "discard what violates the invariant," over different domains, at different times |
| **Centroid-reprojection** — take the centroid of survivors as the next start. **Caveat:** this is Gustafson's informal email shorthand, *not* a documented ubox primitive; his published method keeps and propagates the whole surviving *c-solution* set and refuses to collapse it | **Proof-preserving re-canonicalization** — cut-elimination / normalization re-expressing the carried bundle in the next dialect's canonical form without losing what it carries | **Loosest of the three.** Runtime reprojection picks one representative seed (lossy by construction — which is why Gustafson's own method resists it); compile-time re-canonicalization is by-construction *lossless* for certified passes. Present the slot-correspondence, not an equation |

### How it instantiates the existing discipline

This is a direct instance of *proofs carry through the MiddleEnd, not re-derived
from the final binary.* The conservation invariant is a Tier-≤3 decidable fact:

- **Tier 1** discharges the free part — that the projection operator and the
  momentum/energy functionals are dimensionally and grade sound (zero proof code,
  parametric theorem-for-free), which also guarantees no phantom Clifford grades
  enter the conserved-quantity computation.
- **Tier 2** — a bound on the invariant's admissible range is a QF_LIA/QF_BV
  obligation discharged by an SMT solver in microseconds, surfaced as
  `[<Requires>]`/`[<Ensures>]`, never inline `forall`/theorem.
- **Tier 3** covers any probabilistic/termination flavor of an acceptance or
  widening step. No relational pRHL/Rocq (Tier 4) is required.

The fact does **not** re-derive from the final hardware-specific binary or
netlist. It travels through the MiddleEnd in MLIR's SMT dialect, computed at
design time as a weakest precondition (reportable in the language server as the
engineer types) and re-checked at each lowering edge — the re-check catches a
pass that silently dropped an annotation. Representation choice
(b-posit32+quire vs IEEE vs fixed-point vs FP64+Kahan) is codata on the PSG deferred
to target-binding, so the four demo panes are four bindings of one source with the
contract type-checking identically in every case.

The decidability gate applies in full: symbolic interval evaluation is restricted
to a total, terminating sub-language, and the widening operator must terminate.
That terminating-widening operator is the well-founded measure guaranteeing the
interval fixpoint converges; it bounds the **analysis** and is categorically
distinct from any claim about temporal error growth of the simulated dynamics.

## Three costumes, one mechanism

| Domain | Invariant | The same mechanism | Status |
|---|---|---|---|
| **HPC / three-body** (watchable instance) | A Noether conservation law: total linear momentum exactly zero from rest, plus angular momentum and energy (7 after fixing the barycentric frame; 10 classical integrals before) | ubox in its native habitat: blur to `[x−ULP, x+ULP]`, prune tiles violating the conserved quantities, confine the survivors to the codimension-k invariant manifold. The PSG carries the invariant as a declared `[<Invariant>]` drift sensor. Pruning to the manifold is the *proposed* mechanism for suppressing the off-manifold wrapping-effect blow-up — whether it yields linear growth is Gustafson's open conjecture, Kahan the standing counter-voice | partial |
| **Quantum circuit expression** (readiness story) | Unitarity / norm preservation: the state stays on the Hilbert unit sphere, every gate unitary. The computation is integrity-defined — a result that left the unit sphere is meaningless, exactly as a trajectory that lost momentum is wrong | The same ubox-on-a-manifold pattern with the conservation manifold replaced by the Hilbert-sphere constraint: the carried invariant `R·conj(R) = 1` (the rotor unit-norm constraint already discharged in the ADM paper as a DTS dimensional invariant) generalized to gate unitarity. Representation remains codata deferred to target-binding, so the contract is designed to type-check identically whether lowered to a classical simulator or a quantum substrate | architectural |
| **AI / ADM** (co-equal-with-gradient inversion) | Structural integrity co-equal with the gradient: grade preservation (a declared grade-k weight stays pure grade-k for all steps), rotor unitarity (so equivariance is structural, not statistical), Clifford structural zeros staying zero through training | The quire is ubox pruning in arithmetic clothing: exact accumulation keeps the structurally-zero Cayley entries at exactly zero through training. PHG grade inference is the PSG carrying the invariant — forward-mode dual-number training keeps the grade-k tangent grade-k *by type*; any grade-j contribution is a design-time type error. This is the Grade Invariance proposition and its Sparsity Stability corollary in the ADM paper | partial |

The quantum costume is **architectural throughout** — the corpus is
unitarity-silent; it is *is-designed-for*, built on the demonstrated rotor
unit-norm machinery in the ADM paper, not realized as a quantum lowering target.
The ADM grade-invariance and sparsity-stability results are stated as a **proved
proposition and corollary** in the paper (proof sketch present), so the
design-level guarantee is demonstrated *on paper*; the shipping training stack
that realizes it end-to-end, and the 85–95% sparsity figure (cited from Flash
Clifford, not measured under this substrate), are architectural.

## Why the three-body reversal is the proof-of-life

It makes the abstract claim watchable with zero domain knowledge, and the law is
its own ground truth. A general audience cannot adjudicate a unitarity certificate
or a grade-purity proof, but anyone can watch three bodies launched from rest run
forward, reverse, and either return to their start or visibly miss it. The honesty
is structural: from rest, total linear momentum is exactly zero by physics, so it
needs no external reference to be believed — any nonzero post-reversal value is
pure accumulated rounding and nothing else, which is why the corpus independently
named linear momentum the tightest pure-arithmetic-drift sensor.

The claim must bound the **invariant** (momentum/energy drift stays small), never
the **trajectory** (which provably diverges exponentially in a chaotic regime by
the positive Lyapunov exponent, Nepomuceno). Saying this out loud strengthens
credibility and pre-empts the wrapping-effect objection.

It is the classical rehearsal of the quantum-readiness story because the mechanism
is identical: a quantum computation is integrity-defined exactly as the
conservation demo is — the state must stay on a constraint manifold (Hilbert
sphere there, the conserved manifold here) through every transformation, and both
ask whether the framework can carry that manifold constraint as a certified
invariant toward the substrate.

## The differentiation claim

> We have found no representative implementation, in the standing literature we
> have reviewed, of **integrity-as-carried-certified-invariant**: a structural law
> of the computation — a conservation law, a unitarity constraint, a grade-purity
> invariant — declared at source and propagated through every lowering pass as a
> re-checked decidable fact toward heterogeneous targets, co-equal with the
> optimization objective rather than degrading silently as a side effect of
> optimizing a single scalar.
>
> Adjacent work covers parts of this, and is the result of deliberate scoping, not
> oversight: validated-numerics integrators (Lohner/QR, Taylor models, CAPD,
> VNODE) give guaranteed enclosures within a single numerical method;
> structure-preserving symplectic integrators bound a conserved quantity over long
> times; geometric-algebra ML libraries hardcode initialized sparsity; quantum
> toolchains check unitarity at the circuit level. None we have reviewed carries a
> developer-declared invariant as a first-class object across a general compiler's
> middle-end toward heterogeneous targets, which is the specific claim here.

## Guardrails (so this stays honest)

- **No shipping-present-tense.** The `RealInterval` domain that would
  operationalize the compile-time half is research-grade and unbuilt; Composer
  cannot do ubox today. Architectural verbs for the compile-time half; the runtime
  half is Gustafson's open conjecture.
- **Do not collapse the two boxes.** The whole thesis fails if a reader believes
  the compile-time `RealInterval` encloses the runtime trajectory linearly. Bound
  the invariant, never the trajectory.
- **Not a resolution of a tension.** Conservation as diagnostic and conservation
  as constraint are two jobs side by side; this material is a clearly-labeled
  *third* demonstration that adds, never replaces the honest two-pane drift read.
  One rule: do not measure drift on a path you constrained by projection.
- **Attribute the centroid step** as Gustafson's informal shorthand, contradicting
  his own keep-the-whole-set philosophy.
- **State the count honestly:** 10 classical integrals, 7 only after fixing the
  barycentric frame.
- **Keep the reversibility distinctions intact** and do not let this principle
  borrow strength from a reversibility-implies-conservation slippage.
- **No "tapered precision near zero" anywhere.** The quire's value is exactness of
  accumulation, full stop. Fixed 800-bit quire for b-posit (independent of precision), eS=5.

## The smallest test of the principle

A single conserved quantity surviving **one** lowering edge with a real per-pass
Z3 re-check. Declare total linear momentum as an `[<Invariant>]` on the `State`
type of a minimal leapfrog step in natural units (`G = 1`), with the
functional computed over the b-posit32+quire path. Confirm Tier 1 discharges the
dimensional/grade soundness of the momentum functional for free. Then lower the
PSG across exactly one MiddleEnd dialect-refinement edge that touches the
accumulation, emit the invariant-preservation obligation into the SMT dialect as a
QF_BV/QF_LIA query, and require the solver to return UNSAT on the negation at that
edge. The principle holds at its smallest if the **same** obligation discharges
identically for a CPU binding and an FPGA b-posit binding of that one source —
the conservation fact carried through the lowering rather than re-derived from the
target.

Two explicit non-goals keep the test honest: it does **not** build the
`RealInterval` abstract interpreter (research-grade, out of scope), and it makes
**no** claim about linear-vs-exponential growth of the simulated trajectory (that
is the open runtime conjecture, a different layer). The test validates
compile-time integrity-of-lowering for one invariant across one edge and one
CPU→FPGA target swap. Nothing more.
