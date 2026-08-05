# ThreeBody: Sharpened Demo Design (Gustafson synthesis)

> **Status:** design proposal. ThreeBody `src/` is empty; this supersedes the
> drift in `backgrounder.md` / `design.md` / `simulation-design.md` where they
> conflict. Grounded against John Gustafson's June 2026 correspondence, the
> framework pre-prints, and a web pass over the standing numerical-analysis
> literature (Nepomuceno on interval divergence, Hairer–Lubich backward-error
> analysis, Hayes on shadowing/softening, Gustafson's *The End of Error*).

## The thesis, corrected

ThreeBody is an **honesty harness for arithmetic**, not a verifier of three-body
physics. On a symplectic, time-reversible integrator run from rest, total linear
momentum is identically zero by construction, so any nonzero value it ever reads
is pure accumulated rounding — the cleanest pure-arithmetic-drift sensor
available. The demo swaps **only the number type** across one identical
force-plus-integration path and measures conserved-quantity drift after
forward-then-reverse.

The headline read is the flat line in the b-posit32+quire pane where IEEE drifts,
with the surviving edge located precisely at the un-softened close encounter — in
the cancellation of `qᵢ − qⱼ` and the rounding of `1/r²`, neither of which Kahan
summation touches.

We say out loud what we do **not** claim: the demo bounds the **invariant**
(momentum, energy, angular-momentum drift stay small), never the **trajectory**,
which in a chaotic regime provably diverges exponentially with the largest
Lyapunov exponent. Stating this is the credibility move; it disarms the expert in
the room before they raise a hand.

## What conservation does here: two jobs, not a competition

Conservation laws do two distinct jobs in this demo, side by side, and they never
compete:

- **As a diagnostic** — conserved quantities measure rounding drift on an
  *unconstrained* integrator. The path is left alone, the invariant is read out,
  the flat line is the result. (Panes 1–4.)
- **As a constraint** — Gustafson's ubox method *eliminates* candidate
  state-boxes that violate the invariants and re-projects. The path is actively
  corrected each step. (Optional pane 5.)

Both are legitimate uses of the same seven quantities. The split of jobs **is**
the resolution; there is no conflict to resolve. One bookkeeping rule keeps it
honest:

> **Never measure drift on a path you just constrained by projection.**

Reading momentum-drift on the ubox pane would be circular — that pane forces its
own drift toward zero by construction. Its job is to show what
conservation-constrained integration *buys* (longer coherent evolution,
suppressed off-manifold spread), never to report arithmetic drift. So the two
diagnostic panes stay pristine and the ubox pane is labeled as a demonstration of
a method.

## The panes

| # | Pane | Role | What it shows |
|---|---|---|---|
| 1 | IEEE FP64 | diagnostic (foil) | large, fast-growing reversal residual + invariant drift; the baseline everything is measured against |
| 2 | **b-posit32 + quire** | diagnostic (headline) | small, slowly-growing residual; near-flat invariant line; edge at `qᵢ−qⱼ` and `1/r²` |
| 3 | FP64 + Kahan | diagnostic (**load-bearing control**) | exact compensated summation in the limit; pre-empts "Kahan closes the gap" |
| 4 | fixed-point | diagnostic (anchor) | the only genuinely bit-reversible representation; the limiting case that bounds the comparison |
| — | linear-momentum strip | tightest sensor (spans 1–4) | identically zero from rest → any reading is pure rounding |
| 5 | conservation-pruned ubox | **constraint** (additive, open conjecture) | what conservation-constrained integration buys; Gustafson's proposal, explicitly *not* a drift measurement |

Two overlays:

- **"reversal contract: typechecked ✓" badge** over *both* the IEEE and posit
  panes, above the runtime residual curve. Separates structural reversibility
  (the type, holds for both representations) from numerical reversibility (the
  arithmetic, posit only). Prevents the audience from hearing "typed therefore
  exact."
- **Independent high-precision reference** overlay (a measured illustration, not
  a claimed bound), optionally captioned "neither point is the trajectory" to
  ground Gustafson's discretization motivation.

Pane 3 is not optional. Without the Kahan control the posit advantage is
**unfalsifiable** — a skeptic's first move is "Kahan closes the gap." Run it live
and let the *measured* separation set the claim's strength. The posit edge
survives only where Kahan does not reach: the single subtraction and the
reciprocal.

## Changes against the current docs

1. **Add the linear-momentum drift pane.** `simulation-design.md` instruments
   only energy (l.42) and angular momentum (l.52). From rest, total linear
   momentum is identically zero — the tightest pure-arithmetic-drift sensor, and
   `Numeric_Selection_Implementation.md` §13 already flags it as omitted.
   Gustafson reaches the same seven conserved quantities from the physics side;
   the framework reached linear-momentum-drift from the arithmetic side. They
   agree. **Cheapest high-value change.**
2. **Resolve 2D-vs-3D before the talk.** `simulation-design.md` is "2D only"
   (l.9, l.108), which collapses the count to **4** (1 energy + 2 linear + 1
   scalar angular). If the talk cites Gustafson's **7**, the sim must be 3D (3
   linear + 3 angular + 1 energy). The count and the dimensionality must not
   contradict each other on a slide.
3. **State the count as "7 after fixing the barycentric frame."** The full
   classical problem has **10** algebraic integrals (Bruns 1887, Poincaré 1889:
   1 energy + 3 linear + 3 angular + 3 center-of-mass). Gustafson's 7 drops the 3
   COM integrals, correct only in the barycentric frame. A numerate audience
   member will catch an unqualified "7."
4. **Make the FP64+Kahan control load-bearing, not optional.** (See panes.)
5. **Add the ubox integrator as a clearly-labeled third pane**, additive, never
   substituting for the honest two-pane read. Present as Gustafson's **open
   conjecture** (linear-bound growth) under Kahan's standing rebuttal (the
   wrapping effect drives exponential blow-up, defeatable only at `k^(6N)`
   recompute cost). Carry the surviving **set** forward per the published method;
   flag the centroid-of-survivors step as Gustafson's informal email shorthand,
   **not** a documented ubox primitive — it contradicts the method's
   keep-the-whole-set philosophy.
6. **Remove softening entirely; scope the no-soften choice to this point-mass
   demo.** The un-softened `1/r²` near-singularity is exactly where IEEE
   catastrophic cancellation bites and posit wins; soften it and the curve
   flattens and the demo argues against itself. Do **not** repeat Gustafson's
   "softened sims are totally bogus" as a general claim — Hayes's shadowing work
   shows softening is what makes large collisionless N-body sims reliable. Scope
   it: *"softening would hide the very cancellation we are measuring."*
7. **Scrub the arithmetic drift.** 512-bit quire → **800-bit** (fixed, a vector of 25 32-bit integers, independent of
   precision); reconcile b-posit to **eS=5**, not es=2; **delete** the uncited "39%
   faster decode" claim; **remove** all "tapered precision near zero / golden
   zone" framing (posits taper toward 1.0 and *lose* precision near 0 — the
   quire's value is exactness of accumulation, full stop).
8. **Normalize to natural units `G = 1`** and state the resulting ranges.
   Unnormalized SI `G ≈ 6.7e-11` lands in the wide-dynamic band that selection
   routes to IEEE, not posit. `G = 1` puts `r` near 1, the posit
   high-precision near-unity band. Without normalization the demo argues against
   itself *even with no softening.*
9. **Add an independent high-precision reference trajectory.** Energy is measured
   against the analytically-known `E₀` (good), but absolute trajectory divergence
   needs a reference so the symplectic method's own secular phase drift cannot
   masquerade as arithmetic signal. Use the same quire ruler for both panes when
   reading invariants.
10. **Reframe reversibility as horizon extension, not bit-reversibility.** The
    reversal is live re-computation `Φ⁻¹ = S∘Φ∘S`, not tape replay. Negative
    types *certify* structural completeness of the inverse; they do not nullify
    arithmetic error. Both IEEE and posit type-check the contract identically. Do
    **not** type the velocity flip as `Neg<Velocity>` (that is value-level `−v`, a
    category error; `Neg` rides the adjoint channel and preserves dimension). The
    honest claim is that b-posit+quire holds the reversal longer; neither makes it
    exact.

## Presentation arc (keynote)

Open on **why it is hard**, not the result. State Gustafson's discretization
point as motivation: a coordinate moving 3.4 → 3.5 in one step passes through
every intermediate value, so no single force-evaluation point is legitimate —
the founding premise of validated numerics (Moore, Lohner, Taylor models, CAPD)
and the reason rigorous enclosure exists. Frame the whole talk as an honesty
harness for arithmetic.

Establish the one true, falsifiable claim cleanly: from rest, total linear
momentum is identically zero, so any nonzero reading is pure accumulated
rounding. **Bound the invariant**, and say explicitly that the **trajectory**
provably diverges exponentially in a chaotic regime (positive Lyapunov,
Nepomuceno) and that we do not and cannot claim to bound it. This pre-emptive
concession is the credibility move.

Reveal the four diagnostic panes as four bindings of one source, swapping only
the number type: IEEE drifts, b-posit+quire stays flat, fixed-point is the
bit-exact limit, and run **Kahan live** and let the measured separation set the
claim's strength. Locate the surviving posit edge precisely at the un-softened
close encounter. Show the reversal as horizon extension with the typechecked
badge on both panes.

Only then introduce the ubox pane as a clean split of jobs: conservation as
diagnostic (what we just measured) versus conservation as constraint (what
Gustafson proposes), with the one bookkeeping rule stated plainly and the
linear-bound growth presented as his open conjecture with Kahan as the named
counter-voice.

Close by lifting the ubox shape up one level (see
[`gustafson-synthesis-thesis.md`](gustafson-synthesis-thesis.md)): the
ULP-box + conservation-filter + reprojection pattern is the same *shape* by which
our Program Semantic Graph is **designed to** preserve a computation's integrity
as it lowers. The compiler does not execute the runtime ubox; it is designed to
*emit and certify* it, carrying the conservation invariant as a Tier-bounded fact
through the MiddleEnd so the FPGA b-posit pipeline is **intended to** run the same
conservation-respecting step the source described. End forward-looking and
first-person: rigorous conservation-pruned enclosure is the direction the work
keeps building toward.

## Open questions to settle before the talk

- **2D-with-4 or 3D-with-7?** The sensor panel, the Gustafson "7" citation, and
  the renderer (currently 2D affine) must all agree.
- **Can the ubox pane ship as a runtime numerical method** (buildable inside the
  simulated program) while the *certification* story stays architectural, without
  tripping no-verification-facade? Yes, if the pane is labeled a runtime method
  and the PSG-certification claim uses architectural verbs throughout.
- **Tier-3 seal syntax is OPEN/unbuilt** and is the critical-path language gap
  (see the gap-list). The un-softened `1/r²` kernel falls through Tier-1
  selection (sign-crossing reciprocal yields unbounded pieces — correctly), so
  the buildable path is a developer seal of b-posit32+quire at the force site.
- **Does conservation pruning actually suppress the wrapping-effect blow-up
  enough to recover linear growth on this system?** The apex empirical question
  the ubox pane poses. Genuinely open. Run it offline before the talk so the pane
  shows a *measured* curve, not an asserted one.
- **FPGA lowering is WIP.** The certification claim depends on lowering that is
  not finished. Frame as architectural design intent until demonstrated.
