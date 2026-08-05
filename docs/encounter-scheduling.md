# Encounter Scheduling

> **Status: orientation, not design.** This document records a speculative
> direction and the reasoning that constrains it. Its purpose is to fix a
> *bearing* for later work — what would have to be true, what would falsify it,
> and which claims survive a hostile reviewer — rather than to specify something
> to build now. Nothing here is on the critical path for the 2D demos.
>
> Several claims are recalled without being verified against sources. Those are
> marked. **The Lyapunov section (§4) exists specifically to prevent a tempting
> claim from being made in its wrong form**, and should be read before the idea is
> repeated anywhere public.

## 1. The idea

Close encounters are the expensive regime and the scarce one: they are what
list U hands to the FPGA, and the link budget bounds how many can be initiated
per timestep (see [fpga-transport.md](./fpga-transport.md)).

But encounters are not a surprise. A cluster heading for close proximity is
visible in advance. So:

> Detect upcoming close encounters ahead of time. Precompute them while the FPGA
> is otherwise idle — when no encounter is *in play* — cache the result, and at
> the moment a body needs to follow that path, read it back rather than compute
> it.

The scarce resource is then rescheduled rather than enlarged: encounter work
migrates from the moment of the encounter, when everything competes, into the
quiet intervals before it.

## 2. This is less exotic than it sounds

The scheduling half is established practice in collisional N-body work.
Aarseth's NBODY6/NBODY7 maintain neighbour lists, *predict* which pairs are
heading for close approach, and switch those pairs into regularized coordinates
**before** the encounter occurs — the same Kustaanheimo–Stiefel regularization
already cited in [no-softening-treatment.md](./no-softening-treatment.md).

Anticipating encounters and treating them by a different method is what the
rigorous tradition already does. What is being proposed here is to also move the
*work* in time, not only the *method*.

Time reversal is likewise unremarkable: Newton's equations are symmetric under
`t → −t` with `v → −v`, and symplectic integrators (leapfrog / Störmer–Verlet)
preserve that structurally rather than approximately. Backward integration is the
same integrator with a sign flipped. "Fractional" indexing into a trajectory is
**dense output** — standard integrator machinery for evaluating between steps.

## 3. The correction that decides it: cache maps, not paths

**A cached trajectory is worthless.** The system is chaotic — the positive
Lyapunov exponent is already cited in the demo docs. Precompute a path at lead
time *T* and the actual entry state at *T* is not the one assumed. The cached
path describes a system that did not happen.

What can be cached is the **map**: entry state → exit state.

This works because of a specific structural fact: **a two-body close encounter is
integrable.** Kepler is exactly solvable. The chaos lives in *when* an encounter
occurs and *with what state* — not in the encounter itself. So the object worth
precomputing is not a trajectory through the encounter but a high-order expansion
of the encounter map about a reference trajectory, valid over a bounded region of
entry states.

That has the right cost profile for idle-time work: **expensive to build, cheap to
evaluate.** At use time you evaluate a polynomial instead of integrating.

It also composes with the framework's thesis in a way a cached path never could.
The map's validity region is a **certified enclosure** — "this map holds for
entry states within this box, to this error bound." An entry state outside the
box is a loud condition, not a silently wrong answer.

> **Lead:** in astrodynamics this technique exists as **differential algebra** /
> Taylor-map propagation, used for uncertainty propagation and conjunction
> analysis. Names to check first: Armellin, Di Lizia; Berz for the underlying
> differential-algebra machinery (COSY INFINITY). *Recalled, not verified —
> confirm before citing.* If it holds, the technique is not novel, which is good
> news: it means the hard parts have been worked out and the contribution here is
> the certified-arithmetic substrate underneath it.

## 4. What better arithmetic does and does not buy

There is a tempting claim in the vicinity of this work: *that bounded posits and
the quire extend the Lyapunov time.* **They do not, and the claim should not be
made in that form** — it is the version a physicist will take apart first.

**λ is a property of the system.** The Lyapunov exponent describes how fast two
infinitesimally separated *true* trajectories diverge. It is set by the masses,
the configuration, the energy. No arithmetic changes it. Exact computation does
not slow chaos.

**Precision buys horizon logarithmically.** With injected error `Δ₀` and tolerance
`ε`:

```text
Δ(t) ≈ Δ₀ · e^(λt)        ⟹        t_useful ≈ (1/λ) · ln(ε / Δ₀)
```

Halving `Δ₀` buys `(1/λ)·ln 2`. Concretely, if λ⁻¹ is of order a crossing time,
float64 at ~10⁻¹⁶ affords roughly 36 e-foldings — about 36 crossing times of
usable prediction. Reaching ~10⁻³² affords about 73. **Doubling the precision
roughly doubles the horizon.** Real, useful, and linear in *digits* — not a
change to λ.

**The quire does something better than precision.** Near a close encounter the
dominant error is not representation width but catastrophic cancellation in
`r₁ − r₂` together with rounding accumulated across the force summation.
Ordinary summation error grows like √N to N times machine epsilon; a quire makes
the accumulation **exact** until a single final rounding. That is not a smaller
`Δ₀` — it is an error *mechanism removed*, and it is removed precisely in list U,
where many contributions and sharp cancellation coincide.

**Where the intuition is right.** A float64 simulation near a close encounter
frequently diverges *faster than the true λ*, because cancellation injects error
continuously rather than once. That excess is numerical, not physical, and better
arithmetic removes it. So the **observed** horizon can extend substantially, and
near-singular regimes are exactly where the gap is widest. Stated precisely:

> Arithmetic can eliminate divergence **above** λ. It cannot go **below** λ.
> λ is the floor, and the floor is physics.

**The property that actually makes scheduling work is none of the above.** It is
that interval enclosures do not extend validity — they **locate its end**. A
cached map carries an enclosure that widens as it is propagated; when the width
exceeds tolerance the map has expired, and that is detected rather than assumed.

**Every cached map has a checkable expiry date.** That is what makes banking
idle-time work safe, and it is the same move as FMM's error bound one level up:
rigor purchasing the right to skip computation.

### The claim, in the form that survives review

Not *"posits extend the Lyapunov time."*

Rather: **exact accumulation removes the numerical divergence that dominates near
encounters, and interval bounds tell us precisely how long a precomputed map
remains valid.** Defensible, stronger for the scheduling argument, and it
concedes the point a reviewer would otherwise take.

## 5. Architectural consequence: cache on the FPGA

If the cache lives in FPGA block RAM rather than host-side, the wire carries an
**entry state** and a **result** — never the precomputed intermediate work.

That inverts the constraint in [fpga-transport.md](./fpga-transport.md): the
100BASE-T budget stops gating how many encounters can be *handled* per timestep
and gates only how many can be *initiated*. Whether that is a real win depends
entirely on how compact a Taylor map is relative to the state it replaces —
which is the first thing to measure if this is ever pursued.

## 6. What would have to be true

Falsifiable conditions, in rough order of how likely each is to kill the idea:

1. **Lead time must exceed encounter-detection time.** If encounters are only
   visible a step or two ahead, there is no idle interval to exploit and the
   scheme is pointless. **Measure the distribution of detection lead times before
   anything else.**
2. **Lead time must fall inside the Lyapunov horizon.** §4 bounds it: entry-state
   prediction must remain inside tolerance over the lead interval. Idle time
   beyond that horizon cannot be banked at any precision.
3. **The map must be more compact than the work it replaces.** A Taylor map that
   is larger than the trajectory it encodes buys nothing, and on-chip storage is
   scarce.
4. **Encounters must actually cluster in time.** The scheme is workload smoothing;
   it assumes idle intervals exist. If encounters are uniformly distributed there
   is nothing to smooth.
5. **Map construction must be cheaper than direct integration by enough to pay
   for the machinery.** Otherwise compute the encounter when it happens.

Any one of 1, 2 or 4 failing ends it. They are all cheap to measure on the 2D
demo, and **none require the scheme to be built first.**

## 7. Why record a speculation at all

This document is orientation for the second and third horizons rather than a plan
for the first.

- **Horizon 1** — the 2D demos: conservation (float vs b-posit/quire) and the FMM
  field simulation. Encounter scheduling is *not* part of this and should not
  delay it.
- **Horizon 2** — if FMM's list U becomes the throughput bottleneck, scheduling is
  the direction to look, and §3's map-not-path correction is the form to start
  from rather than rediscovering it.
- **Horizon 3** — the general technique: **certified precomputation.** Build an
  expensive object once, carry a rigorous validity bound with it, and evaluate it
  cheaply until the bound expires. Encounter maps are one instance; conjunction
  analysis and constraint exploration in astrodynamics are structurally the same
  shape — precompute-with-certificate in place of exhaustive exploration.

The value of writing it down now is directional. It says which way to look when
the bottleneck arrives, and it fixes the *form* of the claim before the tempting
wrong version — §4's Lyapunov claim — gets said out loud somewhere it cannot be
retracted.

## 8. Open items

- Measure the encounter-detection lead-time distribution in the 2D demo. This is
  condition 1 and it is cheap.
- Estimate λ⁻¹ for representative configurations; that is the ceiling on bankable
  lead time.
- Verify the differential-algebra / Taylor-map literature (§3) before citing it.
- Compare observed divergence rate under float64 versus b-posit/quire near
  encounters. This is the measurement that substantiates §4's "excess divergence
  above λ" claim, and it is a **result the conservation demo produces anyway.**

## References

- [The Softening Question](./no-softening-treatment.md) — why list U keeps the
  true `1/r²` law; KS regularization.
- [FMM and the Regime Mapping](./fmm-regime-mapping.md) — where list U comes from.
- [FPGA Transport](./fpga-transport.md) — the budget this scheme reschedules.
