# The Softening Question: Why `+h` Is Standard, and Why It Is Wrong Here

> **Status:** internal-rigorous argument notes, for the presentation. Every claim
> on both sides is cited. This artifact is deliberately *separate* from the
> simulation and design docs: those docs never mention `h`, because naming a thing
> in order to reject it re-centers the discussion on it. This is where `h` is named,
> argued, and disposed of — so the other docs do not have to.
>
> **One-line thesis:** softening (`r² + h²` in the denominator) is a legitimate
> *physical* modeling choice for *collisionless* N-body systems and the wrong choice
> for a *point-mass exactness* demonstration — and the honest alternative to it is
> not "remove `h` and hope" but *regularization*, the way celestial mechanics has
> handled close encounters for sixty years.

---

## 1. What softening is

The Newtonian pair force has a `1/r²` singularity as two bodies approach:

```
F = G·m₁·m₂ / (r₁ − r₂)²          # the true force law
```

Softening replaces it with a regularized form that stays finite as `r → 0`:

```
F = G·m₁·m₂ / ((r₁ − r₂)² + h²)   # Plummer softening, h = softening length
```

This is the **Plummer softening** scheme: particles interact as point masses when
far apart and as finite Plummer spheres when they overlap, so the force smoothly
declines to zero at zero separation rather than diverging
([Dehnen 2001, "Towards optimal softening in 3D N-body codes"](https://arxiv.org/abs/astro-ph/0011568);
[overview in Price & Monaghan and the standard codes — see §2](#2-why-it-is-commonly-used-and-this-is-not-a-hack)).
It is built into essentially every production N-body code (GADGET, AREPO, GIZMO).

---

## 2. Why it is commonly used — and this is *not* a hack

This is the part the demo must **steelman**, not strawman. The standard
justification is *physical*, not numerical convenience:

A cosmological or galactic N-body simulation is trying to model **collisionless**
dynamics — the smooth Vlasov–Poisson / collisionless Boltzmann evolution of a
continuous phase-space distribution. The simulation samples that *continuous*
distribution with a finite number of discrete particles. Two such particles
passing close would scatter off each other — but that scattering is an **artifact
of the discretization**, not physics: the real system is smooth and has no such
close pairs. Softening suppresses exactly this artificial two-body scattering,
making the discrete particle potential behave more like the smooth density field
it is meant to represent
([Softening in N-body simulations of collisionless systems, Dehnen 2001](https://arxiv.org/abs/astro-ph/0011568);
[Power et al. on optimal ε and convergence](https://arxiv.org/abs/astro-ph/0201544)).

In that context softening is **correct**, and the literature even shows it is
*reliability-essential*. Hayes' shadowing analysis
([Hayes 2003, *Shadowing-based reliability decay in softened n-body simulations*, ApJL 587, L59](https://arxiv.org/abs/astro-ph/0211128))
establishes an *explicit* relationship between the decay of shadowable particles
and the timestep `h_t`, the softening `ε`, and the particle count `N`. The result
that matters here: shadowability **degrades sharply if softening is made too
small** — below ~0.2 of the local mean inter-particle separation — and a large
fraction of particles stay shadowable over ~100 crossing times *only* when
softening is in an appropriate range and particles move ≤ ⅓ of a softening length
per step. In other words, for a collisionless system, **removing softening makes
the simulation *less* trustworthy, not more.**

> **Presentation guardrail.** Do *not* say "softened simulations are bogus" as a
> general claim. They are correct and necessary for the problem they model. Hayes
> is the citation that proves softening earns its place there. The demo's claim is
> *scoped*, and the scope is the next section.

---

## 3. Why it is wrong *here* — the scope that decides everything

The three-body demo is **not** a collisionless system. It is three **point masses**.
There is no continuous distribution being sampled, no artificial two-body
scattering to suppress, no graininess to smooth. The close encounter the demo is
built around is not a discretization artifact — **it is the physics.** The `1/r²`
singularity is real, and it is the precise place where the demo's actual subject —
catastrophic cancellation in `r₁ − r₂` and the arithmetic that survives it — does
its work.

So softening here does two unacceptable things:

1. **It changes the problem.** `G·m₁·m₂/((r₁−r₂)² + h²)` is a *different force law*
   from `G·m₁·m₂/(r₁−r₂)²`. A simulation of the softened law is solving a different
   set of equations and cannot be reported as a result about gravity. Whatever
   conservation behavior it shows is conservation of the *softened* dynamics.
2. **It hides the very effect being measured.** The demo exists to show where IEEE
   floating point loses significance and exact (b-posit + quire) accumulation does
   not — and that loss happens *at the close encounter*, in the differencing of
   nearly-equal large coordinates. Soften the encounter and the singularity
   flattens, the cancellation never gets sharp, and the demo argues against
   itself: it would be measuring an arithmetic difference at a point where the
   arithmetic no longer matters.

The scope line, stated for the talk: **softening would hide the very cancellation
we are measuring** — and for a point-mass system there is no collisionless
justification to invoke it in the first place.

---

## 4. The honest alternative is *regularization*, not "remove `h` and hope"

This is the section that defeats the obvious rebuttal — *"you took out softening,
so your integrator blows up at the singularity; you've just traded one problem for
a worse one."* The answer is that the rigorous celestial-mechanics tradition has a
*correct* way to handle the real singularity that is neither softening nor naïve
integration: **regularization** — a coordinate/time transformation that removes the
singularity analytically rather than smoothing it away physically.

The canonical instance is the **Kustaanheimo–Stiefel (KS) transformation**, which
lifts the gravitational two-body problem into four dimensions where it becomes a
*harmonic oscillator* — turning the close encounter into a smooth, exactly
integrable arc rather than a numerical cliff
([KS regularization of the three-body problem, Lega & Guzzo 2024, *Physica D*](https://arxiv.org/abs/2206.07022)).
KS regularization **reduces the order of the pole at the singularity** and lets a
close encounter be integrated *stably and accurately with the true force law
intact* — the singularity is handled by the *mathematics*, not deformed out of the
*physics*.

So the corrected treatment is a **two-part** answer, and both parts are citable:

- **Do not deform the force law.** Keep `1/r²`. (Against softening: §3.)
- **Handle the real singularity correctly.** Either regularize the close encounter
  (KS, the celestial-mechanics standard) or — the framework's own substitute —
  bound the *domain of validity* with a non-zero lower bound on `r` (a `r ≥ r_min`
  annotation that makes the singular regime an *explicit, loud* condition rather
  than a silently-softened one), and carry the near-singular state honestly through
  exact accumulation (the quire) and interval enclosure rather than rounding it
  away. This is the framework's "bound the domain, don't deform the law" position,
  and it is the principled cousin of regularization: where KS removes the pole by
  transformation, the framework refuses to *pretend the pole is not there*.

The point for the audience: **the choice is not "softening vs. blow-up."** It is
"softening (changes the physics) vs. regularization (preserves the physics and
integrates the real singularity correctly)." The demo is on the side the rigorous
literature has been on since Kustaanheimo & Stiefel (1965).

### 4.1 What this looks like in code

> **Syntax note.** The representation-seal and range-law forms below are
> *illustrative of the handling pattern*, not settled Clef. The Tier-3 seal syntax
> is the project's open critical-path language gap (see `gustafson-synthesis-gaplist.md`);
> these blocks show the *shape* of the no-softening treatment, with the proposed
> surface marked. The dimensional types (`float<m>`, `float<N>`) and the actor/quire
> machinery are the parts that lower today.

First, the thing the demo refuses — softening folds an unphysical constant *into the
arithmetic*, where the compiler cannot see it as anything but a number:

```fsharp
// REJECTED. The `+ h*h` silently changes the force law. Worse for the compiler:
// the singularity is now *gone* — the denominator can never reach zero, so the
// expression is trivially bounded, Tier-1 selection sails through, and nothing on
// the program graph ever records that a singular regime existed. The physics and
// the structural fact are both erased at the source.
let forceSoftened (m1: float<kg>) (m2: float<kg>) (r: Vector3<m>) (h: float<m>) : Vector3<N> =
    let r2 = Vector3.dot r r + h * h            // <- the lie: r² + h²
    (G * m1 * m2 / r2) * Vector3.normalize r
```

The no-softening treatment keeps the true law and handles the real singularity as
*structure on the graph*, in three parts:

```fsharp
// 1. The true force law. No `h`. The `r₁ − r₂` subtraction stays exact because the
//    accumulation runs in the quire; the seal (part 3) is what puts it there.
let force (m1: float<kg>) (m2: float<kg>) (p1: Vector3<m>) (p2: Vector3<m>) : Vector3<N> =
    let d  = p1 - p2                            // the cancelling subtraction, carried exactly
    let r2 = Vector3.dot d d                    // r²  — genuinely → 0 at a close encounter
    (G * m1 * m2 / r2) * Vector3.normalize d

// 2. The honest substitute for `+h`: bound the DOMAIN, not the law.
//    `r_min` is a minimum-approach distance supplied by a range law / companion
//    attribute (never a hard-coded magic constant). It does not enter the
//    arithmetic — it is a carried invariant the graph certifies and that fires a
//    LOUD diagnostic if the simulation ever drives `r` below it, instead of a
//    softening term silently absorbing the violation.
[<Invariant>]                                   // proposed surface
let minimumApproach (s: ThreeBodyState) : float<m> =
    s.PairwiseSeparations |> Seq.min            // must stay ≥ r_min, declared elsewhere

// 3. The Tier-3 seal. Because `force` has no nonzero lower bound on `r` in dataflow,
//    intrinsic (Tier-1) representation selection CANNOT bound it and falls through —
//    correctly, and loudly. The developer seals the force site to b-posit32 + quire,
//    which is what makes the un-softened kernel lower without waiting on the
//    (research-grade, unbuilt) real interval domain.
let forceSealed (m1: float<kg>) (m2: float<kg>) (p1: Vector3<m>) (p2: Vector3<m>) : Vector3<N> =
    seal<posit32, Quire> (force m1 m2 p1 p2)    // proposed seal surface
```

What the three parts buy, structurally: the singularity is **not** smoothed out of
existence, so it survives onto the program graph — as a Tier-1 fall-through that
fails *loudly* (the compiler says "range unobservable" rather than fabricating a
floor), as a carried `r ≥ r_min` domain invariant, and as a Tier-3 seal that routes
the node to exact (b-posit32 + quire) arithmetic. That last fact is *why* the
close-encounter regime binds to the exact-arithmetic target in the first place — the
routing is a consequence of the un-softened node's structure, not an arbitrary
assignment. Softening would have deleted all three before the graph was ever built.

---

## 5. The citation discipline (why this artifact exists)

The entire force of this argument is that **both sides are cited**:

- The **error case** is cited to its *own* proponents and steelmanned — softening
  is physically motivated for collisionless systems
  ([Dehnen 2001](https://arxiv.org/abs/astro-ph/0011568);
  [Power et al. 2003](https://arxiv.org/abs/astro-ph/0201544)) and even
  reliability-essential there ([Hayes 2003](https://arxiv.org/abs/astro-ph/0211128)).
- The **scope** that makes it wrong here is a *category* distinction
  (collisionless distribution vs. point masses), not an assertion of error.
- The **corrected form** is cited to the regularization literature
  ([KS regularization, Lega & Guzzo 2024](https://arxiv.org/abs/2206.07022); the
  underlying Kustaanheimo–Stiefel 1965 transformation), so "no softening" is shown
  to be the *rigorous* tradition's position, not a contrarian one.
- The **chaos caveat** that keeps the demo honest — trajectory divergence is
  expected and is *not* a softening failure — is the positive-Lyapunov result
  already cited in the demo docs (Nepomuceno), and the shadowing framing is Hayes.

A skeptic's strongest move ("everyone softens; you removed it; your sim is the
broken one") is answered on every clause by a citation, and the answer is not
"Gustafson says so" — it is "softening models a *different* system; for *this*
system the rigorous treatment is regularization, and here is the celestial-
mechanics literature that has used it for sixty years."

---

## 6. Open items / to verify before the talk

- **Kustaanheimo–Stiefel 1965 primary citation.** The original is Kustaanheimo &
  Stiefel, *J. Reine Angew. Math.* 218 (1965), 204–219. Confirm the exact reference
  for a slide; the [Lega & Guzzo 2024](https://arxiv.org/abs/2206.07022) paper is
  the verified modern source used here for the *three-body* application.
- **Whether the demo actually regularizes or only domain-bounds.** §4 offers two
  corrected paths (KS regularization vs. `r ≥ r_min` domain annotation + exact
  accumulation). Decide which the demo claims. If it does *not* implement KS, the
  honest framing is the domain-bound path, and the KS literature is cited as "the
  rigorous tradition this aligns with," not "what we did."
- **GADGET/AREPO/GIZMO specific softening citations.** Named here as the production
  codes that soften; if a slide names them, cite each code's release paper directly
  rather than relying on the secondary characterization.
- **Hayes quantitative thresholds.** The "~0.2 mean separation" and "⅓ softening
  length per step" figures are from the verified abstract of
  [Hayes 2003](https://arxiv.org/abs/astro-ph/0211128); confirm against the full
  text before putting exact numbers on a slide.
