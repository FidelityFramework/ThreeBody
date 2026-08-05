# FMM and the Regime Mapping

> **Status:** design note, build-oriented. Argues that the Fast Multipole Method
> is the algorithm that makes ThreeBody's two demos *one* implementation, and that
> its regime decomposition already matches the heterogeneous target mapping rather
> than having to be forced onto it.
>
> Companion to [The Softening Question](./no-softening-treatment.md): that document
> argues what **not** to do at the singularity. This one argues for the algorithm
> that puts the singularity in its own box, so the treatment has somewhere to live.

## 1. Why FMM and not Barnes-Hut

Barnes-Hut is the algorithm everyone reaches for, and for this project it is
disqualified by a single property: **its error is difficult to bound.**

That is not a performance objection. It is the same objection the softening
document makes about `+h²`. A simulation whose approximation error cannot be
bounded can be *shown*, but it cannot be *claimed*. ThreeBody's entire thesis is
that the framework preserves what other treatments quietly discard, and an
unbounded approximation in the far field would hand a reviewer the obvious
rebuttal: *you kept exactness where the camera is pointed and threw it away
everywhere else.*

FMM (Greengard & Rokhlin, 1987) is O(N) with a **derivable error bound**, and the
bound has the right shape:

```text
error  ~  ( r / |z| ) ^ (p+1)          r = cluster radius
                                       |z| = distance to evaluation point
                                       p = truncation order
```

Exponential in the truncation order `p`, and decaying with separation. That gives
a knob (`p`) with a *proof* attached rather than a heuristic (`θ`) with a
reputation attached.

> **The framing that matters for this project:** a bounded approximation is a
> **certificate**. It is the same currency as the interval enclosures and the
> representation-selection tiers elsewhere in the design — an argument the
> compiler and the reviewer can both check. Barnes-Hut's `θ` is a knob you tune
> until it looks right. Those are different kinds of claim.

## 2. The regime decomposition *is* the target mapping

The project already assigns physics regimes to processors. FMM produces the same
partition from the algorithm's own structure, which means the mapping stops being
an assertion and becomes a consequence.

| FMM structure | what it computes | target | why |
| --- | --- | --- | --- |
| **List U** — adjacent boxes and self | direct pairwise, no approximation | **FPGA** — b-posit + quire | the only regime FMM refuses to approximate; where cancellation is sharp |
| **List V** — well-separated at same level | M2L flip, bounded error | **GPU** — wide SIMT over many boxes | uniform, parallel, error-bounded |
| **Lists W / X** — size-mismatched neighbours | expansion evaluated per-particle | **GPU / NPU** | irregular; the adaptive cases |
| **Upward / downward passes** | M2M, L2L tree traversal | **CPU** | sequential, pointer-chasing, low arithmetic intensity |
| **List Y** | nothing — already handled higher | — | |

**The critical property: list U is exactly where softening is normally applied.**
Plummer softening exists to tame the `1/r²` singularity between close pairs, and
close pairs are precisely what lands in list U. FMM isolates that regime
*structurally*, as the one place it declines to expand — and hands back a small,
bounded set of pairs.

That is where the un-softened force law and the Tier-3 seal belong:

```text
list U  →  force (no h)  →  seal<posit32, Quire>  →  FPGA
```

The seal now has a **principled scope** rather than a hand-drawn one. It is not
"seal the close encounters because they look important"; it is "seal list U,
because list U is by construction the set the approximation refuses to cover."

## 3. One implementation, two demos

This is the practical payoff and the reason to build FMM first rather than as a
later optimisation.

- **The conservation demo** (three point masses, float vs b-posit/quire) is the
  **N=3 limit of the field demo**: the tree is degenerate, every pair is in list
  U, and no expansion is ever formed. Same kernel, same seal, same force law.
- **The visual demo** (hundreds of thousands of particles) is the same code with a
  populated tree, where lists V/W/X carry the far field.

So the conservation result is not a separate program that happens to share a
theme — it is the field simulation with the approximation path unused. That is a
considerably stronger story than two demos side by side, and it means the exact
kernel is exercised by *both* from day one.

## 4. Starting in 2D is the right call, and not only for scope

2D is not merely the cheaper option. It is qualitatively simpler mathematics,
because the 2D potential is a **complex logarithm** and that collapses several
awkward steps at once:

- The potential is `log|z − zᵢ|`, the real part of the complex `log(z − zᵢ)`.
  Distance computations become complex arithmetic.
- **Force comes from the derivative for free.** By Cauchy–Riemann, the complex
  derivative `Φ'(z)` gives the force as a complex number by negating the real
  component — no separate gradient, no finite differencing, no accuracy loss in
  recovering the force from the potential.
- Both expansions become plain power series in `z`, and the operators (§5) are
  polynomial manipulation.

In 3D none of that survives. The expansions become **spherical harmonics**,
translation operators grow substantially more intricate, and the elegant
complex-derivative route to force is gone. Kernel-independent variants exist and
are also more work.

> **Decision recorded:** the project targets **2D** for both the conservation
> demo and the visual field demo. 3D with a moving camera is a distinct piece of
> work and is out of scope here.

## 5. The operators to build, in order

FMM is five operators plus direct evaluation. Each is small; the algorithm is
their composition. Build and test them in this order — each is independently
checkable against direct summation, which is the whole testing strategy.

| # | operator | does | test against |
| --- | --- | --- | --- |
| 1 | **direct** | pairwise force in list U | analytic two-body |
| 2 | **P2M** | particles → multipole expansion at leaf | direct, outside the cluster radius |
| 3 | **M2M** | shift child multipoles to parent centre | P2M computed directly at the parent |
| 4 | **M2L** | *flip* a multipole into a local expansion | direct summation from the source box |
| 5 | **L2L** | shift a parent local down to children | L2L result vs. re-derived local |
| 6 | **L2P** | evaluate local expansion at particles | direct |

Two properties worth knowing before writing them, because they determine how much
error bookkeeping is needed:

- **M2M and L2L introduce no additional error.** Truncated at `p` terms, the
  shifted expansion is exact to the same `p` terms. They can be applied at any
  depth without degradation.
- **M2L does.** Knowing the first `p` multipole coefficients is *not* sufficient
  to obtain the first `p` coefficients of the flipped local expansion, so the flip
  incurs its own error term. It is still exponentially decreasing with separation,
  but it is the operator whose error must be tracked.

A further subtlety worth recording, because it is easy to get wrong and produces
a slow, correct-looking simulation: **expansions are centred on the box centre,
not the cluster's centre of mass.** Centre-of-mass centring seems natural and
inflates the radius of convergence badly whenever a lone particle sits opposite a
dense corner. Box centring keeps the convergence radius tied to the box size,
which is the quantity the separation criterion is expressed in.

## 6. Uniform first, adaptive second

The uniform-grid FMM assumes roughly even particle distribution and is
substantially simpler. Gravitational fields **clump**, so the uniform version will
not hold up on the real distribution — but it is the right first target, because
it exercises all six operators with none of the size-mismatch bookkeeping.

Adaptive FMM (Carrier, Greengard & Rokhlin, 1988) adds lists W and X purely to
handle boxes of mismatched size:

- **W**: a source box much smaller than the target — cannot form a useful local
  expansion, so evaluate its multipole per target particle.
- **X**: the dual — a source box much larger than the target — so build the
  target's local expansion from the source's individual particles.

Both lists are bounded in size independently of N, which is what preserves O(N)
under clustering.

## 7. Why the bound is load-bearing beyond this demo

A bounded approximation is not only an integrity argument. It is what licenses
*not doing work*.

The general principle the demos exhibit: **when a region's contribution can be
rigorously bounded, the work in that region can be discarded rather than
performed.** FMM discards pairwise interactions across the far field because the
expansion error is provably small. Interval enclosures elsewhere in the framework
discard candidate regions because the enclosure proves them dominated. Both are
the same move — rigor purchasing the right to skip computation — and it is a
sharper claim than "we go faster."

This is why the error bound, not the O(N), is the property to lead with. Speed
from an unbounded approximation is a trade. Speed from a bounded one is a
consequence.

## 8. Open items

- **Truncation order `p`.** The bound gives error as a function of `p` and
  separation; pick `p` from a target accuracy rather than by eye, and record the
  derivation. `p = 10` was visually indistinguishable in the reference material,
  which is a starting point, not a justification.
- **Where the separation criterion is enforced.** Barnes-Hut's `θ` becomes FMM's
  list construction. The criterion should be a stated invariant, not a constant
  buried in tree code.
- **Interaction with the timestep and the `r ≥ r_min` domain bound.** The
  softening document's domain invariant lives in list U. Confirm the interaction:
  a close encounter that drives `r` below `r_min` should fire the loud diagnostic
  regardless of which FMM list the pair landed in.
- **Per-timestep payload across the FPGA link.** List U's size determines how much
  close-encounter state crosses to the sidecar each step. See
  [fpga-transport.md](./fpga-transport.md) — this is the number that decides
  whether 100BASE-T bandwidth binds.

## References

- Greengard, L. & Rokhlin, V. (1987). *A fast algorithm for particle simulations.*
  The original FMM paper.
- Carrier, J., Greengard, L. & Rokhlin, V. (1988). *A fast adaptive multipole
  algorithm for particle simulations.* The adaptive extension; lists W and X.
- [The Softening Question](./no-softening-treatment.md) — why list U keeps the
  true `1/r²` law.
- [FPGA Transport](./fpga-transport.md) — the link that carries list U's work.
