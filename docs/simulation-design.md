# Three Body: Simulation Design

## Overview

The Three Body demo runs two identical gravitational simulations side by side from the same initial conditions. One simulation uses b-posit arithmetic on an FPGA sidecar for close-encounter force computation. The other uses IEEE FP64 on the CPU. The audience watches the two paths diverge over time, then watches a time-reversal test prove which path was more accurate.

## Initial Conditions

Three equal-mass bodies placed at the vertices of a right triangle on a 2D gravitational plane, with zero initial velocity. All motion stays in the initial plane.

This configuration is chosen deliberately:

- **Equal masses** eliminate any asymmetry in the physics. The only asymmetry comes from the triangle geometry.
- **Right triangle** (not equilateral) breaks symmetry immediately. An equilateral triangle is a stable Lagrangian solution where the bodies simply orbit the center of mass. A right triangle produces close encounters quickly.
- **Zero initial velocity** means total energy is purely gravitational potential (negative), guaranteeing a bound system. No body escapes to infinity.
- **The triangle aspect ratio is a tuning parameter.** A more elongated right triangle (one short leg, one long leg) produces an early two-body near-collision with the third body arriving later. A 1:1 isosceles right triangle has all three bodies converging at roughly the same time. Both are valid; the elongated version is easier for an audience to follow because events happen sequentially.

## Dual-Pane Layout

Two simulation panes displayed side by side, running in lockstep from identical initial conditions and the same timestep:

**Left pane (b-posit):** Close-encounter force computations route to the FPGA sidecar running b-posit32 arithmetic with 800-bit quire accumulation. All other computation (bulk forces, orchestration) runs on CPU/GPU. FPGA board LEDs blink when pairs stream through the b-posit pipeline.

**Right pane (IEEE FP64):** All computation runs on CPU. Close encounters use standard IEEE double precision. No FPGA involvement. The board sits dark.

Both panes share the same viewport (zoom level, center point) so the visual comparison remains valid at all times. Bodies are rendered with colored trails showing their trajectory history.

## Dynamic Viewport

The three-body dynamics produce dramatic scale changes. Close encounters bring bodies within fractions of the initial separation, while slingshot ejections fling a body to 100x or 1000x the initial separation. The viewport must adapt.

**Automatic zoom.** Each frame, compute the bounding box of all three bodies in the b-posit (reference) pane, add padding, and set the viewport to encompass everything. Both panes use this viewport.

**Smooth transitions.** The viewport tracks toward the target bounds using exponential interpolation rather than snapping. The zoom feels fluid, not jarring.

**Picture-in-picture inset.** When the viewport is zoomed out to track a distant body, the close-encounter region becomes too small to see. A magnified inset window in the corner shows the close-encounter pair at high zoom. The inset appears when close encounters are active (FPGA LEDs blinking) and disappears when all bodies are at comparable separations. This preserves the "see the FPGA working" story even at wide zoom.

## Validation: How We Prove Correctness

The demo must answer "how do you know the b-posit simulation is more correct, not just different?" without asking the audience to take anything on faith.

### Energy Conservation (Quantitative)

Total energy (kinetic plus potential) is exactly conserved in a closed gravitational system. This is a consequence of Noether's theorem, not a modeling choice.

The initial energy E_0 is computed from the chosen initial conditions: masses, positions, and velocities that the developer defined as exact values. For three bodies this is six terms (three kinetic, three potential). Computed with the quire, E_0 is exact.

At each timestep, total energy is recomputed from the current positions and velocities. For both simulations, this measurement uses the FPGA quire (same ruler for both paths). Any deviation from E_0 is entirely numerical integration error. The simulation whose energy line stays flatter is, by definition, more accurate.

The energy conservation graph runs alongside both simulation panes throughout the demo as a live quantitative measure.

### Angular Momentum Conservation (Secondary)

Total angular momentum is independently conserved. Same logic: compute from known exact initial conditions, measure at each timestep, drift is error. This provides a second independent validation channel.

### Time-Reversal (Visual Proof)

This is the demonstration that requires zero domain knowledge to evaluate.

1. Show the starting right triangle. "These are our initial conditions."
2. Run forward, dual pane, trails accumulating. Energy graph running.
3. Freeze both simulations after N timesteps.
4. Reverse all velocities. "Now we reverse time."
5. Run backward, same number of steps, same timestep.
6. Overlay final positions on the starting triangle for both panes.
7. The b-posit pane's bodies land back on (or very near) the triangle vertices. The FP64 pane's bodies are somewhere else.

> One path found its way home. The other got lost. The difference is arithmetic.

Same physics, same timestep, same number of steps. One path used exact accumulation for close encounters. The other did not. The starting triangle is its own ground truth.

## Demo Arc

A single run follows a natural narrative arc:

**Act 1: Setup.** Show the starting right triangle in both panes. Explain the initial conditions. Point out the FPGA board. Start the simulation.

**Act 2: Divergence.** The simulations run in lockstep. Trails accumulate. At first the panes look identical. Gradually, the trails begin to separate. The energy graph shows one line drifting while the other stays flat. The PnP inset appears during close encounters, FPGA LEDs blink for the left pane only. The audience watches the gap grow.

**Act 3: Scale.** A slingshot ejection sends a body on a wide orbit. The viewport zooms out smoothly. The PnP inset shows the remaining tight binary. The scale change is dramatic and visceral.

**Act 4: Reversal.** When the divergence between panes is visually obvious, freeze both simulations. Reverse velocities. Run backward. The b-posit simulation returns to the starting triangle. The FP64 simulation does not.

**Act 5: Overlay.** Freeze-frame. Superimpose the trail histories from both panes. Show the energy conservation graph side by side. The single overlay image captures the entire precision argument and works as a screenshot in a slide deck or paper.

**Reset.** Return to initial conditions. Optionally vary the triangle aspect ratio or masses for a second run with different pacing. Each run tells the same story.

## Reversibility Horizon

Every chaotic system has a reversibility horizon: the number of forward timesteps after which accumulated numerical error makes accurate reversal impossible. The Lyapunov exponent of the system determines how quickly small errors grow, and the arithmetic precision determines how small those errors start.

The demo is tuned to find a timestep count N where the b-posit simulation still reverses accurately but the IEEE FP64 simulation does not. This is the dramatic sweet spot. Too few steps and both return home (no story). Too many and neither returns (also no story). The right N depends on the initial triangle geometry and the timestep size, both of which are tuning parameters.

### Precision Spectrum

Three points on the precision continuum are relevant:

**IEEE FP64.** 15-16 significant digits. Fixed-width significand. Catastrophic cancellation in close-encounter subtraction. Hits the reversibility horizon earliest.

**B-posit32 with quire.** Tapered precision favoring the golden zone where scientific values cluster. 800-bit quire provides exact intermediate accumulation. The only rounding occurs at final conversion. Extends the reversibility horizon well beyond IEEE FP64. The bounded regime field (rS=6) enables a 39% decode speed advantage over IEEE float32 in hardware.

**Unbounded posit.** Standard posit format without the regime bound. Would extend the reversibility horizon further still, because the variable-length regime field provides even more dynamic range. The tradeoff is computational: unbounded posits require sequential decoding (a long barrel-shift chain), which is exactly the hardware bottleneck that b-posit's bounded regime was designed to eliminate. For the demo's purposes, unbounded posits are too slow to run at interactive framerates on the FPGA.

The demo sits at the b-posit point on this spectrum: the best available balance of precision and hardware performance. The audience sees the gap between IEEE float and b-posit. The design documentation notes the gap between b-posit and unbounded posit for completeness, but the demo does not attempt to show it.

## Technical Notes

**2D only.** All computation and rendering is in a 2D plane. No camera rotation, no perspective projection. The viewport is a 2D affine transform: translate to center the bounding box, scale to fit. Negligible rendering cost.

**Trails.** Each body leaves a colored trail (ring buffer of recent positions, rendered as a polyline). Trail length is a tuning parameter. Longer trails show more history but can clutter the display at wide zoom.

**Timestep.** Fixed timestep for both simulations (not adaptive). This eliminates any "different timestep" objection and keeps the simulations in lockstep. The timestep size is a tuning parameter that affects how quickly divergence becomes visible.

**Rendering.** Platform.Display (Wayland pixel buffer). Pure Clef functions compiled to native by Composer. No external rendering library. Body positions as filled circles, trails as polylines, PnP inset as a clipped sub-viewport, energy graph as a line plot. Bitmap font for labels.

**Actor topology.** Each simulation pane is its own actor subtree under Prospero. The b-posit pane has an FpgaBPosit actor; the FP64 pane does not. Both share the CpuOrchestrator for timestep coordination. Telemetry actor reads hardware counters and updates the energy graph and processor metrics.
