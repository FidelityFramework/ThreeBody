# Interval Arithmetic as a Runtime Representation

> **Status:** design note, front-burner. Grounded against the Composer/clef
> sources, the posit and numeric-selection docs, and the negative-types paper
> (two verification passes, June 2026). Motivated by John Gustafson's
> discretization point: a coordinate that sweeps an interval over a step cannot
> honestly be evaluated at any single point. This asks whether the framework can
> carry that interval *as a runtime value*, lowered to hardware — distinct from
> the compile-time interval *analysis* it already has.

## The distinction that governs everything

There are two unrelated "intervals" in the framework, at two different times.
Conflating them is the central trap.

| | Compile-time interval **analysis** | Runtime interval **type** |
|---|---|---|
| What it is | a static `{Min, Max}` / `RealInterval` over the *set of values a node could take across inputs* | a `{lo, hi}` pair that *travels with one concrete value as it computes* |
| When | design time; **disappears at lowering** | runtime; **runs on hardware** |
| Purpose | select a representation, infer a width, prove a bound | carry the unknown honestly through a computation |
| In the framework | `IntervalAnalysis.fs` (integer, ships) + `RealIntervalDomain` (real, research-grade unbuilt) | a single one-line claim in `posit-arithmetic.md:363`, **unbuilt** |

The numeric-selection post (`deferred-inference`) and the Composer
implementation guide are entirely about the **left column**. John's point, and
the ubox method, need the **right column**. They are not the same idea and the
left one does not imply the right one.

## What lowers today, and what does not

The honest accounting splits cleanly into the *container* and the *arithmetic*.

**The container lowers today.** Struct-wrapped numeric types are native and reach
hardware now:

- `Posit32` is `[<Struct>] { Bits: uint32 }`; the whole Posit8/16/32/64 family
  uses the same pattern (`posit-arithmetic.md:56–98`).
- The **Quire** is a *multi-field* runtime numeric struct — `Quire32` is 8×uint64,
  `Quire64` is 32×uint64 — and it lowers through the MLIR struct dialect, on FPGA
  via `hw.struct_create` / `hw.struct_extract` (`posit-arithmetic.md:114–142`;
  `Composer/.../RecordPatterns.fs:40–73`).
- SRTP dispatch over these is in `BasicOps` (`posit-arithmetic.md:246–280`).

A two-field `Interval<posit32> = { lo: Posit32; hi: Posit32 }` is structurally
*the same lowering pattern* as these. The **Quire is the precedent** — a real,
multi-field, posit-native numeric struct that already lowers to fabric. (Note:
dual numbers are **not** the precedent. Forward-mode AD carries tangents as
*coeffects*, design-time, not as a runtime `{primal, tangent}` struct — so the
"if dual numbers lower, intervals lower" analogy does not hold. The Quire one
does.)

**The arithmetic does not lower today — and this is the real gap.** Interval
arithmetic requires **directed (outward) rounding**: `lo` must round toward −∞ and
`hi` toward +∞ at every operation, or the pair stops being a sound enclosure.
There is no per-operation rounding-mode control anywhere in the framework today —
no `RoundTowardNegative`, no directed-rounding facility. The Composer guide lists
"outward-rounded FP interval arithmetic — endpoints must round outward to remain a
sound superset" as a *required, unbuilt* component of the real interval domain
(`Numeric_Selection_Implementation.md:143`).

So the precise status is: **the interval container is buildable on existing
struct-lowering; the interval arithmetic is gated on directed rounding, which is
the genuine unbuilt primitive.** The `posit-arithmetic.md:363` sentence asserting
intervals "can be expressed as pairs of posits" is correct about the container and
silent about the rounding — it is speculative architecture, not demonstrated
lowering.

## The division of labor with the quire (and an overclaim to fix)

`posit-arithmetic.md:363` currently argues the quire makes valids/ubounds
*unnecessary*. That is **too broad**, and it is worth correcting in our own docs
because it papers over exactly the gap that makes a runtime interval worth having.
The quire eliminates **accumulation error** — the compounding of rounding over a
long sum of products. It does **not** address two other things:

1. **Single-step catastrophic cancellation.** `qᵢ − qⱼ` with `qᵢ ≈ qⱼ` loses
   significant digits in *one* subtraction. No accumulator helps; the digits are
   gone before accumulation begins. This is the close-encounter site in the
   three-body demo and the place the FP64+Kahan control cannot reach either.
2. **The epistemic unknown.** An ODE step whose true trajectory swept `[3.4, 3.5]`
   never knew which path it took. That is not an arithmetic error at all; it is
   missing information, and only carrying the interval makes it honest.

The clean framing, then, is a **division of labor, not redundancy**:

> The **quire** makes accumulation exact, so you never need an interval to *bound*
> a sum. The **interval** makes the unknown explicit, so you carry what the quire
> cannot fix — the single cancelling subtraction, and the path you did not
> observe.

They compose: a force MAC accumulates in the quire (exact), while the position it
reads is carried as an interval (honest about the step). One handles the
*accumulation* axis, the other the *epistemic* axis. The doc should be amended
from "the quire makes intervals unnecessary" to "the quire removes the
accumulation motivation for intervals; the single-step and epistemic motivations
remain, and a runtime interval type addresses those."

## Where it sits structurally: a runtime witness

This is the genuinely new category, and it is worth naming because the four-tier
verification model is silent on it. An interval carried at runtime that *provably
contains* the true value is an **epistemic witness** — the program computed a
value *and* computed that it lies in `[lo, hi]`. That is structurally a **runtime
refinement type** (a value paired with a proof of a predicate), but discharged at
runtime rather than compile time.

Against the four tiers:

- It is **not** a Tier-1 free theorem (those are parametric, not epistemic).
- It is **not** a Tier-2 QF_LIA bound (that is *static* range analysis on
  exponents, the compile-time interval).
- It is a **runtime witness to a Tier-2-shaped property** — the dynamic companion
  to the static bound, carried in the value rather than proved over the program.

The framework has no current category for "runtime witnesses to compile-time
properties." That is the design opening here, and it is coherent and orthogonal to
everything already in scope, not a replacement for any of it.

## What this is *not* (keeping the axes straight)

- **Not reversibility.** The negative-types/reversibility machinery (`Φ⁻¹ = S∘Φ∘S`,
  live re-computation, the adjoint channel) is a property of the **map**. The
  interval's unknown-trajectory point is an **epistemic** property of the forward
  step. The corpus correctly keeps these separate, and so must we — a reversible
  step still does not know the path it swept. The two *compose* (reversibility as
  the map structure, intervals as epistemic witnesses on the states it maps) but
  neither implies the other, and "adiabatic therefore interval" is a category
  slip to avoid.
- **Not the compile-time interval domain.** Building `Interval<posit>` as a
  runtime type does **not** require the research-grade `RealIntervalDomain`
  abstract interpreter. They share the word and nothing else.

## Build sketch (honest about the gap)

1. **`Interval<r>` as a two-field struct** `{ lo: r; hi: r }` for `r ∈ {posit,
   b-posit, fixed}` — lowers today on the Quire's struct-lowering path.
2. **Directed-rounding capability** as a per-target fact (the real gap): on a CPU,
   the rounding-mode register; on an FPGA, two datapaths laid down with the
   rounding direction fixed in fabric (which suits the framework — the direction
   is a *design* property there, not a runtime mode switch); on a posit target,
   whether the hardware exposes directed rounding or it must be emulated. This
   slots into the same three-valued capability gate (`Native | Emulated |
   Unavailable`) numeric selection already uses, and **fails loudly** where
   directed rounding is unavailable — never a silently-unsound enclosure.
3. **Outward-rounded interval operations** (`+`, `×`, the sign-crossing `recip`
   that returns 0–2 pieces for a denominator straddling zero — the same `1/r²`
   case the no-softening force forces) compiled as the operations *on* the pair,
   given the rounding capability.
4. **Compose with the quire**: accumulate exactly, carry the operand interval.

The critical-path item is **directed rounding**, not the type. That is the one
sentence that turns "hand-wavy in the docs" into a real lowering story.

## Relationship to Gustafson's constructs

Gustafson's own runtime-interval types from the unum work are the reference point,
and our docs cite them without defining them, which should be fixed:

- A **valid** is an interval bounded by two posits — the range of possible true
  values. `Interval<posit>` above *is* a valid, by another name.
- A **ubound** is a unum interval guaranteed to contain the true value (a compact
  tagged form).

So this avenue is, honestly, **valids lowered to Fidelity hardware** via the
existing struct path plus directed rounding — not a new type system, which is
exactly what `posit-arithmetic.md:363` claims, now with the gap named. The open
question worth putting to Gustafson is whether posit-pairs-plus-directed-rounding
is the right shape, or whether the ubound's tagged form buys something the bare
pair does not.
