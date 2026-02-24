# The Three Body Problem: From Fiction to Precision

## The Cultural Moment

In 2008, Liu Cixin published *The Three-Body Problem*, the first volume of a trilogy that would become one of the most celebrated works of science fiction in any language. The novels brought hard science fiction to a global audience, culminating in the 2024 Netflix adaptation that introduced millions of viewers to concepts from orbital mechanics, computational physics, and the fundamental limits of prediction.

We owe Liu Cixin and the production teams at Netflix, Plan B, and Yoozoo a genuine debt of gratitude. Their work cultivated interest in physics and mathematics at a scale that no textbook or lecture series could match. When a Netflix viewer asks "is the three-body problem really unsolvable?" they have taken the first step toward understanding computational complexity, numerical methods, and the relationship between mathematical models and physical reality. That curiosity is a gift to STEM education, and this project exists in part because of the cultural space they created.

## Where the Story Stops

The novels and series frame the three-body problem as fundamentally chaotic, depicting a system so unpredictable that an advanced civilization cannot determine when their next stable era will arrive. This is not wrong. The three-body problem *is* chaotic. It *is* sensitive to initial conditions. Small differences in starting values produce large differences in outcomes over time, and Poincaré proved in the 1890s that no single algebraic formula produces a general solution for arbitrary initial conditions.

The fiction tells this part of the story well, and it makes for extraordinary drama.

> But the story stops here, and the physics continues.

### The Question the Fiction Doesn't Ask

What Poincaré proved is that a certain line of mathematical inquiry was closed. He did not prove that the problem is unknowable. The absence of a closed-form general solution does not mean we cannot compute trajectories. It means we compute them numerically, step by step, and the quality of that computation depends on the precision of our arithmetic.

"Chaotic" does not mean "unknowable." It means that the precision of your answer depends on the precision of your arithmetic. That distinction is where this project begins.

### Numerical Integration Works

In practice, physicists and aerospace engineers solve n-body gravitational problems routinely. The solar system is an n-body problem, and we navigate spacecraft through it with extraordinary precision. The key is numerical integration: compute forces at each timestep, update positions and velocities, repeat.

The challenge is not whether this works. The challenge is *how accurately* it works over long time horizons, and that challenge is fundamentally a question of arithmetic precision.

### The Real Villain Is Rounding Error

Every floating-point operation introduces a small rounding error. In a gravitational simulation, these errors accumulate at every timestep. Over millions of timesteps, the accumulated error can exceed the actual forces being computed.

IEEE 754 floating-point arithmetic is the standard used by virtually every processor manufactured today. It has specific, well-understood precision limits. A 64-bit IEEE float (double precision) carries about 15-16 significant decimal digits. When two bodies approach closely, the gravitational force involves subtraction of nearly-equal large numbers, catastrophically canceling significant digits. 

> This is arithmetic with choices attached and consequences that result.

The three-body problem is not unpredictable because physics is mysterious. It becomes unpredictable when your arithmetic runs out of precision. The fiction gave us the drama of chaos. The question it left on the table is: "can we solve this *precisely enough* with the arithmetic available to us?"

## Enter B-Posit Arithmetic

Most readers will not have encountered this yet. John Gustafson has spent decades studying the relationship between numerical formats and computational accuracy, and his posit number format has been a known alternative to IEEE 754 floats in the numerical computing community for several years. What is newer, and not yet widely available, is the *b-posit* refinement.

B-posit is a draft specification (Jonnalagadda, Thotli, and Gustafson, currently in review as of early 2026) that solves the main practical objection to posit hardware. Standard posits use a variable-length regime field that requires sequential decoding, making them slower than IEEE floats in silicon. B-posit bounds that regime field to a fixed maximum of 6 bits, which turns the decode into a simple parallel MUX operation. The result is that a 32-bit b-posit decoder is 39% *faster* than an IEEE float32 decoder in benchmarked hardware. This changes the conversation from "posits are theoretically better but impractical" to "posits are faster, smaller, and more accurate for the workloads that matter."

We are building Three Body on b-posit because we believe the draft will support that next revision of the posit standard. This is an inside track position. If b-posit becomes the reference format, this project will be among the first hardware implementations. We like that "flex" to go with a concrete demonstration of b-posit's value proposition.

For our purposes, two properties of b-posit arithmetic are directly material:

**The quire accumulator.** B-posits define an 800-bit fixed-width accumulator (the quire) that provides *exact* accumulation. A sum of products computed in the quire loses zero precision at any intermediate step. The only rounding occurs at the final conversion back to a b-posit value. Every intermediate result is exact. This is a mathematical property of the format, not an engineering aspiration.

**Tapered precision.** Within what the spec calls the "Golden Zone" (roughly 10^-20 to 10^20 for 32-bit b-posit), the format provides more significand bits than an IEEE float of the same width. Near 1.0, where most scientific values cluster, the advantage is substantial. IEEE floats maintain more uniform precision at extreme magnitudes. The formats have different strengths in different regimes, and this demo is designed to show exactly where those boundaries fall.

For gravitational simulation, these properties have a concrete and measurable consequence. Close-encounter force computations routed through b-posit arithmetic with quire accumulation produce results that are provably more accurate than the same computation in IEEE FP64, while using fewer bits. The proof is simple: run the simulation forward, reverse it, and see which path returns to its starting position. The b-posit path comes home. The IEEE path does not.

## Why This Demo Matters

Our design for the "Three Body" demo is not simply about solving a physics problem. It is a case study in something more fundamental: *workload-aware heterogeneous compute*.

### The Turning Point

Different regimes of the same problem have different computational characteristics. Bulk pairwise force calculation is massively parallel and tolerant of FP32 precision. Far-field approximation is a sustained inference workload. Close-encounter precision demands custom arithmetic. Orchestration is sequential and control-heavy.

> No single processor architecture is optimal for all four. 

A GPU excels at the bulk computation but cannot implement custom number formats. An FPGA can implement b-posit arithmetic but cannot match GPU throughput on uniform parallel work. An NPU is power-efficient for inference but not designed for general computation. A CPU can do everything but excels at nothing in particular.

The right answer is to use all four simultaneously, with each processor handling the solution space elements it was designed for.

### What the Application Surfaces

Two identical simulations run side by side from the same initial conditions: three equal-mass bodies at the vertices of a right triangle, released from rest. The left pane routes close-encounter force computation through b-posit arithmetic on the FPGA. The right pane uses IEEE FP64 on the CPU. Same physics, same timestep, same code.

The audience watches both panes evolve. At first, the trails are identical. Gradually, they diverge. The left pane's FPGA board blinks as close-encounter pairs stream through the b-posit pipeline. The right pane's close encounters go through standard float arithmetic and nobody touches the board.

When the divergence is visually clear, both simulations freeze. All velocities reverse. Both simulations run backward the same number of steps. The b-posit pane's bodies return to the starting triangle. The FP64 pane's bodies do not.

> One path found its way home. The other got lost.

The starting triangle is its own ground truth. The audience does not need to understand energy conservation or Noether's theorem. They can see that one simulation is reversible and the other is not, and the only difference is the arithmetic used for close encounters.

### The Compiler Story

All four processor targets are compiled from a single Clef codebase by the same compiler (Composer). The Alex middle-end emits target-appropriate MLIR dialects: `arith/scf/llvm` for CPU, `gpu/rocdl` for GPU, `aie/aievec` for NPU, and `handshake/hw/comb` for FPGA. The application developer writes ML-family source code that describe the function, the display characteristics for the front end of the application, and the mechanics for shared data contracts and failover scenarios. 

This is the thesis of the entire Fidelity framework: one language, one compiler, any processor. The Three Body demo is the first application that exercises all four targets simultaneously, supervised by a single actor topology, with workload placement decisions made ***at*** runtime ***without*** a managed runtime.

### The Reversibility Horizon

Every chaotic system has what we call a reversibility horizon: the number of forward steps you can take and still reverse accurately back to your starting point. Beyond that horizon, accumulated numerical error exceeds the precision needed to retrace the path. The toothpaste does not go back in the tube.

IEEE FP64 hits this horizon relatively early. B-posit arithmetic, with its quire accumulation preserving exactness at every intermediate step, extends the horizon considerably further. The demo is tuned to find the sweet spot where b-posit still returns home and IEEE float does not.

But b-posit is not the end of the story. Standard (unbounded) posit arithmetic, where the regime field has no fixed-width bound, would extend the reversibility horizon further still. The tradeoff is computational: unbounded posits require sequential decoding (the very problem b-posit was designed to solve), making them impractical in hardware at the speeds the demo requires. B-posit's bounded regime field is what makes the 39% decode speed advantage possible. It is a deliberate engineering choice: sacrifice some of the format's theoretical precision ceiling in exchange for hardware that actually runs faster than IEEE floats.

The precision spectrum is real and continuous: IEEE float, then b-posit, then unbounded posit, each extending the horizon at increasing computational cost. This demo sits at the b-posit point on that spectrum because it represents the best balance of precision and performance that current hardware can deliver.

### A New Era

We believe this represents a meaningful shift in how systems workloads will be structured. The era of "write for one processor, hope the hardware is fast enough" is giving way to workload-aware targeting where the compiler and runtime cooperate to place computation on the silicon best suited for it.

The three-body problem is not unsolvable. It is a question of precision, and precision is a question of arithmetic. The fiction raised the question. The physics gives us the tools to answer it. This demo shows exactly how far those tools reach today, and where the next generation of numerical formats will take us.
