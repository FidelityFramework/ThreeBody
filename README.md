# Three Body

Heterogeneous compute demo: one Clef codebase, four processor architectures.

A gravitational simulation where physics naturally decomposes into distance regimes,
each mapped to the processor architecture best suited for it:

| Regime | Processor | Why |
|--------|-----------|-----|
| Close encounters (~0.1%) | FPGA (b-posit arithmetic) | Lossless quire accumulation where IEEE FP64 fails |
| Medium distance (~90%) | GPU (SIMT) | Massively parallel FP32 force computation |
| Far field (~10%) | NPU (dataflow) | Neural surrogate inference, low power |
| Orchestration | CPU | Timestep integration, regime classification |

All four targets compiled from the same source by [Composer](https://github.com/speakez-tools/Composer)
via the Alex middle-end and MLIR backends.

## Project Structure

```
docs/       Design documents
src/        Clef source
```

## Key Technologies

- **Clef** -- Concurrent language targeting heterogeneous compute
- **B-posit arithmetic** -- Bounded posit format (es=2) with a 512-bit quire (`n²/2` for posit32) for lossless accumulation
- **Prospero/Olivier** -- Actor supervision across all four processors
- **BAREWire** -- IPC and wire protocol connecting actors and the FPGA sidecar over Layer 2 Ethernet
- **Platform.Display** -- Native Wayland rendering (no WebView)

## Hardware

- ASUS ROG Z13 Flow (Strix Halo: Zen 5 + RDNA 3.5 + XDNA 2)
- Digilent Arty A7-100T (FPGA sidecar over Layer 2 Ethernet, RJ45)

> **Why Layer 2 Ethernet.** Latency dominates this workload: close-encounter
> regimes hand work to the FPGA and need the result back inside a timestep, so
> round-trip cost matters more than raw bandwidth. Raw Layer 2 frames over RJ45
> — bridged with eBPF/wBPF, no IP stack in the path — give the lowest latency
> available short of PCIe, and PCIe is not on offer between this host and this
> board. USB-C, used earlier in the design, does not serve the role.

## License

[MIT](LICENSE)
