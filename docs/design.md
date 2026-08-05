# Three Body: Heterogeneous Compute Demo

## Overview

Gravitational simulation demonstrating heterogeneous compute across four processor
architectures from a single Clef codebase. The name evokes the popular Liu Cixin
novels and Netflix series while the actual simulation handles hundreds of thousands
of bodies with physics that naturally decomposes into distance regimes.

## Four-Way Decomposition

| Regime | Fraction | Precision | Parallelism | Processor |
|--------|----------|-----------|-------------|-----------|
| Close encounters | ~0.1% | Critical | Low, streaming | FPGA (b-posit) |
| Medium distance | ~90% | Standard FP32 | Massive, uniform | GPU (SIMT) |
| Far field | ~10% | Approximate OK | Sustained inference | NPU (dataflow) |
| Orchestration | 1 thread | N/A | Sequential | CPU |

### GPU: Bulk Force Computation
Vast majority of pairwise interactions. Standard FP32, massively parallel.

### NPU: Far-Field Approximation
Trained neural surrogate approximating Barnes-Hut tree for distant body groups.
Streaming, sustained, low-power inference.

### FPGA: Close Encounters with B-Posit Arithmetic
The force law is the true `G·m₁·m₂/(r₁−r₂)²`. Near the close-encounter singularity,
the subtraction `r₁−r₂` of nearly-equal large coordinates is where IEEE FP64 loses
significance, and exact accumulation does not. B-posit arithmetic (eS=5) with a
fixed 800-bit quire (a vector of 25 32-bit integers, independent of precision) provides lossless accumulation of the pairwise-force
dot products — the mechanism that survives an FP64+Kahan control exactly at the
`qᵢ−qⱼ` and `1/r²` sites where Kahan cannot reach.

**The singularity is carried, not smoothed.** Because the force kernel has no nonzero
lower bound on `r` in dataflow, intrinsic (Tier-1) representation selection cannot
bound it and *correctly falls through* — and the demo's contract is that this
fall-through is **loud**: Composer emits an unobservable-range diagnostic rather than
ever fabricating a floor. The honest substitute for a fabricated floor is a
*domain-of-validity* bound — a minimum-approach `r ≥ r_min` invariant supplied by a
range law or companion attribute, never a hard-coded constant — which bounds the
*domain* without deforming the *law*. The close-encounter node is then sealed
(Tier-3) to b-posit32+quire at the force site. This sealed, domain-bounded singular
node is *why* the close-encounter regime binds to the exact-arithmetic target: the
regime routing in the table above is a consequence of this node's structure, not an
arbitrary assignment. (On the deliberate refusal to add a softening term to the
denominator, and why that refusal is the rigorous celestial-mechanics position
rather than a contrarian one, see [`no-softening-treatment.md`](no-softening-treatment.md).)

### CPU: Orchestration
Simulation loop, timestep integration, regime classification, tree construction,
visualization dispatch.

## Actor Topology

Each processor's workload is an Olivier actor, supervised by Prospero:

```
Prospero (Supervisor -- OneForOne restart strategy)
+-- CpuOrchestrator   actor  ->  tree build, timestep, regime classification
+-- GpuBulkForce      actor  ->  medium-distance FP32 force computation
+-- NpuFarField       actor  ->  neural surrogate inference
+-- FpgaBPosit        actor  ->  close-encounter b-posit pipeline
+-- Telemetry         actor  ->  hardware counter collection + display
```

Fault tolerance: FPGA link loss -> actor dies -> Prospero restarts on
reconnection. Simulation degrades gracefully, then recovers automatically.

## Dimensional Types

```clef
type [<Measure>] kg
type [<Measure>] m
type [<Measure>] s
type [<Measure>] N = kg * m / s^2

type Body = {
    Mass:     float<kg>
    Position: Vector3<m>
    Velocity: Vector3<m/s>
}
```

Zero runtime cost. Errors caught at design time.

## Native Rendering

Platform.Display (Wayland pixel buffer), not WebView. Pure Clef functions compiled
to native by Composer:
- Body positions as dots, colored by regime
- Halos on close-encounter bodies (visual FPGA correlation)
- Per-processor telemetry panels (htop-like)
- Energy conservation graph (b-posit vs FP64 drift)

## Hardware

- ASUS ROG Z13 Flow: Strix Halo (Zen 5 + RDNA 3.5 + XDNA 2), 64GB unified LPDDR5X
- Digilent Arty A7-100T: USB sidecar for b-posit pipeline
- Arch Linux (Omarchy), kernel 6.18+

## Compilation Targets

```
Alex emits MLIR ->
  CPU:   arith/scf/llvm  -> mlir-opt -> LLVM -> native binary
  NPU:   aie/aievec/aiex -> MLIR-AIE -> Peano/LLVM -> XCLBIN
  GPU:   gpu/rocdl       -> MLIR GPU -> LLVM AMDGPU -> HSA code object
  FPGA:  handshake/hw/comb -> CIRCT -> Verilog -> Vivado -> bitstream
```

## Related Projects

- **Composer**: Compiler (Alex middle-end, MLIR backends)
- **Fidelity.Platform**: Display, Console, Compute, Perf subsystems
- **BAREWire**: IPC/memory/wire protocol connecting actors and the FPGA sidecar
- **Warhol**: Three-target predecessor (CPU/GPU/NPU, no FPGA)
