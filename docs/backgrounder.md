# The Three Body Problem: From Fiction to Precision

## The Cultural Moment

In 2008, Liu Cixin published *The Three-Body Problem*, the first volume of a trilogy that would become one of the most celebrated works of science fiction in any language. The novels brought hard science fiction to a global audience, culminating in the 2024 Netflix adaptation that introduced millions of viewers to concepts from orbital mechanics, computational physics, and the fundamental limits of prediction.

We owe Liu Cixin and the production teams at Netflix, Plan B, and Yoozoo a genuine debt of gratitude. Their work cultivated interest in physics and mathematics at a scale that no textbook or lecture series could match. When a Netflix viewer asks "is the three-body problem really unsolvable?" they have taken the first step toward understanding computational complexity, numerical methods, and the relationship between mathematical models and physical reality. That curiosity is a gift to STEM education, and this project exists in part because of the cultural space they created.

## Where the Fiction Departs from the Physics

The novels and series frame the three-body problem as fundamentally mysterious and chaotic, depicting a system so unpredictable that an advanced civilization cannot determine when their next stable era will arrive. This makes for extraordinary storytelling. 

> It is also, in important ways, ***not*** accurate.

### The Problem Is Not Unsolvable

The gravitational three-body problem asks a straightforward question: given three masses with known positions and velocities, predict their future trajectories. The answer is that this is not unsolvable. It is not even particularly mysterious. What it is, precisely, is *sensitive to initial conditions* and *not expressible in closed-form general solutions*.

These are different statements with different implications:

**No closed-form general solution** means there is no single algebraic formula that produces the answer for arbitrary initial conditions. This was proven by Poincaré in the 1890s, and it ended a specific line of mathematical inquiry. It did not end our ability to compute trajectories.

**Sensitive to initial conditions** means that small differences in starting values produce large differences in outcomes over time. This is the hallmark of chaotic systems. But "chaotic" does not mean "unknowable." It means that the precision of your answer depends on the precision of your arithmetic.

### Numerical Integration Works

In practice, physicists and aerospace engineers solve n-body gravitational problems routinely. The solar system is an n-body problem, and we navigate spacecraft through it with extraordinary precision. The key is numerical integration: compute forces at each timestep, update positions and velocities, repeat.

The challenge is not whether this works. The challenge is *how accurately* it works over long time horizons, and that challenge is fundamentally a question of arithmetic precision.

### The Real Villain Is Rounding Error

Every floating-point operation introduces a small rounding error. In a gravitational simulation, these errors accumulate at every timestep. Over millions of timesteps, the accumulated error can exceed the actual forces being computed.

IEEE 754 floating-point arithmetic is the standard used by virtually every processor manufactured today. It has specific, well-understood precision limits. A 64-bit IEEE float (double precision) carries about 15-16 significant decimal digits. When two bodies approach closely, the gravitational force involves subtraction of nearly-equal large numbers, catastrophically canceling significant digits. 

> This is arithmetic with choices attached and consequences that result.

The three-body problem is not unpredictable because physics is mysterious. It becomes unpredictable when your arithmetic runs out of precision. The truth of mathematical accuracy is more interesting than a fantasy narrative: the question is not "can we solve this?" but "can we solve this *precisely enough* with the arithmetic available to us?"

## Enter B-Posit Arithmetic

Most readers will not have encountered this yet. John Gustafson has spent decades studying the relationship between numerical formats and computational accuracy, and his posit number format has been a known alternative to IEEE 754 floats in the numerical computing community for several years. What is newer, and not yet widely available, is the *b-posit* refinement.

B-posit is a draft specification (Jonnalagadda, Thotli, and Gustafson, currently in review as of early 2026) that solves the main practical objection to posit hardware. Standard posits use a variable-length regime field that requires sequential decoding, making them slower than IEEE floats in silicon. B-posit bounds that regime field to a fixed maximum of 6 bits, which turns the decode into a simple parallel MUX operation. The result is that a 32-bit b-posit decoder is 39% *faster* than an IEEE float32 decoder in benchmarked hardware. This changes the conversation from "posits are theoretically better but impractical" to "posits are faster, smaller, and more accurate for the workloads that matter."

We are building Three Body on b-posit because we believe the draft will support that next revision of the posit standard. This is an inside track position. If b-posit becomes the reference format, this project will be among the first hardware implementations. We like that "flex" to go with a concrete demonstration of b-posit's value proposition.

For our purposes, two properties of b-posit arithmetic are directly material:

**The quire accumulator.** B-posits define an 800-bit fixed-width accumulator (the quire) that provides *exact* accumulation. A sum of products computed in the quire loses zero precision at any intermediate step. The only rounding occurs at the final conversion back to a b-posit value. Every intermediate result is exact. This is a mathematical property of the format, not an engineering aspiration.

**Tapered precision.** Within what the spec calls the "Golden Zone" (roughly 10^-20 to 10^20 for 32-bit b-posit), the format provides more significand bits than an IEEE float of the same width. Near 1.0, where most scientific values cluster, the advantage is substantial. IEEE floats maintain more uniform precision at extreme magnitudes. The formats have different strengths in different regimes, and this demo is designed to show exactly where those boundaries fall.

For gravitational simulation, these properties have a concrete and measurable consequence. Close-encounter force computations routed through b-posit arithmetic with quire accumulation produce results that are provably more accurate than the same computation in IEEE FP64, while using fewer bits. The energy drift graph over a long simulation run tells this story visually. 

> Where IEEE FP64 gradually loses energy conservation, the b-posit path maintains it.

## Why This Demo Matters

Our design for the "Three Body" demo is not simply about solving a physics problem. It is a case study in something more fundamental: *workload-aware heterogeneous compute*.

### The Turning Point

Different regimes of the same problem have different computational characteristics. Bulk pairwise force calculation is massively parallel and tolerant of FP32 precision. Far-field approximation is a sustained inference workload. Close-encounter precision demands custom arithmetic. Orchestration is sequential and control-heavy.

> No single processor architecture is optimal for all four. 

A GPU excels at the bulk computation but cannot implement custom number formats. An FPGA can implement b-posit arithmetic but cannot match GPU throughput on uniform parallel work. An NPU is power-efficient for inference but not designed for general computation. A CPU can do everything but excels at nothing in particular.

The right answer is to use all four simultaneously, with each processor handling the solution space elements it was designed for.

### What the Application Surfaces

The Three Body visualization is designed to make the heterogeneous compute story *visible*:

- **Body halos** appear when particle pairs enter the b-posit precision regime, showing the audience exactly which computation has moved to the FPGA.
- **Per-processor telemetry panels** display real-time utilization, power draw, and throughput for each processor, making the workload distribution tangible.
- **Regime classification** is shown as color coding on the particles themselves, so the audience sees the simulation transpose in real time.
- **Energy conservation graph** provides the scientific result: b-posit quire arithmetic maintains conservation where IEEE float degrades.

The FPGA dev board sits on the desk, connected by USB-C, with its LEDs blinking as pairs stream through the b-posit pipelines. An audience would see the computation move to custom arithmetic hardware and return. We'll use motion graphics and blinking LEDs on the FPGA board for illustration, but the key is the math itself.

### The Compiler Story

All four processor targets are compiled from a single Clef codebase by the same compiler (Composer). The Alex middle-end emits target-appropriate MLIR dialects: `arith/scf/llvm` for CPU, `gpu/rocdl` for GPU, `aie/aievec` for NPU, and `handshake/hw/comb` for FPGA. The application developer writes ML-family source code that describe the function, the display characteristics for the front end of the application, and the mechanics for shared data contracts and failover scenarios. 

This is the thesis of the entire Fidelity framework: one language, one compiler, any processor. The Three Body demo is the first application that exercises all four targets simultaneously, supervised by a single actor topology, with workload placement decisions made ***at*** runtime ***without*** a managed runtime.

### A New Era

We believe this represents a meaningful shift in how systems workloads will be structured. The era of "write for one processor, hope the hardware is fast enough" is giving way to workload-aware targeting where the compiler and runtime cooperate to place computation on the silicon best suited for it.

The three-body problem is not unsolvable. It is a question of precision. With the right arithmetic, on the right hardware, managed by software that understands both, the problem not only becomes tractable - it practically becomes trivial. The fiction raised the question, and the truth of our demo will answer it.
