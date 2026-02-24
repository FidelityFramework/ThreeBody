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
Gravitational force near 1/r^2 singularity. IEEE FP64 loses precision exactly where
it matters most. B-posit arithmetic (bounded regime rS=6, eS=5) with 800-bit fixed
quire provides lossless accumulation. B-posit32 decoder is 39% faster than IEEE
float32 decode via MUX-based parallel decode (no LBC, no barrel shifter).

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

Fault tolerance: FPGA USB-C disconnect -> actor dies -> Prospero restarts on
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
- **BAREWire**: IPC/memory protocol connecting actors and USB-C sidecar
- **Warhol**: Three-target predecessor (CPU/GPU/NPU, no FPGA)
