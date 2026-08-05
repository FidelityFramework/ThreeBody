# FPGA Transport: Why Layer 2 Ethernet, Not USB

> **Status:** design rationale. Records *why* the FPGA sidecar link is raw Layer 2
> over RJ45 rather than the board's USB port, so the decision is not re-litigated
> from scratch later. Confidence is marked per claim; the load-bearing numbers are
> flagged for confirmation against primary sources before they appear in a talk.

## The decision

The Arty A7-100T sidecar communicates with the host over **raw Layer 2 Ethernet
frames on RJ45**, bridged on the host side with eBPF/XDP. No IP stack in the data
path.

USB is **not** used for the data path. It remains what it already is on this
board: the programming and console channel.

## Why not USB — the mechanism, not the folklore

The reflexive objection to USB is "it has a stack." True but imprecise, and the
precise version is stronger.

**The Arty's USB port is not a data path.** The board's own bindings say so:
`ArtyA7_100T.Bindings.clef` defines `uart_tx` and `uart_rx` as *"USB-UART TX from
FPGA"* / *"RX to FPGA"*, and HelloArty's monitor reads `/dev/ttyUSB1`. The
connector fronts an FTDI bridge presenting JTAG on one channel and a UART on the
other. Artix-7 has no hard USB device controller and the board carries no ULPI
PHY, so "USB" here means **a serial port tunnelled over USB** — not bulk transfer.

That stacks three latency sources, none of which are bandwidth problems:

| source | cost | can it be tuned away? |
| --- | --- | --- |
| UART serialisation | per-byte, with start/stop framing | only by raising baud, which has a hard ceiling |
| USB host scheduling | transfers wait for a host-polled slot on 125 µs microframe boundaries | no — the device cannot shorten its own wait |
| FTDI latency timer | small transfers buffer until the timer expires | floor is 1 ms; **default is 16 ms** |

The third dominates and is the reason this is structural rather than tunable. For
a request/response smaller than the bridge's internal buffer, **the latency timer
*is* the round-trip time.** Its floor of 1 ms is already an order of magnitude
worse than the alternative.

## Why Layer 2 wins

At 100 Mbit/s, a 64-byte frame is **5.12 µs** of wire time. PHY latency is
sub-microsecond. On the host, XDP runs at the driver before `skb` allocation, so
the kernel-side cost is single-digit microseconds. A round trip is plausibly
**tens of microseconds**, dominated by FPGA-side processing rather than by
transport.

Against a 1 ms floor on the USB path, that is one to two orders of magnitude, and
it is the *shape* of the number that matters: the Ethernet path's cost is
proportional to the work, while the USB path's cost is a fixed toll charged
whether the payload is four bytes or four hundred.

**Latency is the binding constraint for this workload.** A close-encounter regime
hands work to the FPGA and needs the answer back inside a timestep. Round-trip
cost matters more than raw bandwidth, which inverts the usual link comparison.

## The honest counterpoints

Recorded so the decision can be revisited on evidence rather than re-argued from
memory.

1. **USB 2.0 High Speed has more raw bandwidth** — 480 Mbit/s against 100BASE-T's
   100 Mbit/s. But that ceiling requires a real USB device controller this board
   does not have. Through an FTDI UART it is unreachable, so the comparison is
   theoretical.
2. **PCIe would beat both outright.** Artix-7 has GTP transceivers, but the Arty
   A7 exposes no PCIe connector, so it is not available on this hardware.
3. **Ethernet is not free.** It needs a 10/100 MAC in fabric, where the FTDI
   bridge is already on the board and costs no logic. Real work, but well-trodden.

## The constraint to watch: bandwidth, not latency

100BASE-T is roughly **12.5 MB/s**. The latency argument is settled; the bandwidth
one is not, because it depends on a number nobody has computed yet:

> **Open item — compute the per-timestep payload.** How many bytes of
> close-encounter state cross the link each timestep? A few hundred bodies of
> b-posit state is comfortable. If it grows, the wire binds before the deadline
> does, and *that* would be the finding that reopens this decision.

This is the number to measure early. It is also the number that would justify
PCIe-class hardware if the demo outgrows the Arty.

## Claims to confirm before a talk

- **FTDI latency timer: 16 ms default, 1 ms floor.** This is the load-bearing
  figure in the whole argument. Confirm against the FT2232H datasheet and the
  driver documentation.
- **Arty A7 Ethernet PHY is 10/100, not gigabit.** Affects the bandwidth headroom
  above, not the latency conclusion. Confirm against the Arty A7 reference manual.
- **XDP round-trip figures.** The single-digit-microsecond host-side claim is
  general knowledge about the XDP path, not a measurement on this hardware. Worth
  a bench number before it goes on a slide.

## A possible second use for USB

Not a data path, but potentially a *payload* path: if the project home-rolls
bitstream loading to bypass OpenFPGALoader, the JTAG channel is where that would
live. Configuration is throughput-tolerant and latency-indifferent — exactly the
profile USB serves well and Layer 2 does not especially. The two links would then
have clean, separate roles rather than competing for one.
