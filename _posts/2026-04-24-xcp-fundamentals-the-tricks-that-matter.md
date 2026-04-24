---
layout: post
title: "XCP Fundamentals: The Tricks That Matter"
date: 2026-04-24
categories: [xcp, calibration, automotive, embedded]
tags: [xcp, asam, calibration, ecu, daq, a2l, motorsport]
excerpt: "A working engineer's notes on the XCP protocol. The concepts you need, and the quiet details that only show up when you read the spec twice."
---

Most introductions to XCP stop at the surface. It's a protocol for reading
and writing ECU memory, it has a master and a slave, it runs on CAN or
Ethernet. All true, all insufficient. The interesting parts of XCP (the
ones that actually bite you during implementation) live in the details
that casual summaries skip: why the identification field is shaped the way
it is, why DAQ lists aren't just "periodic transmissions", why the
transport-agnostic design forces specific choices in your state machine.

This post is my working notes on those details. It's written as if I'm
sitting next to a colleague who already knows CAN and embedded systems, and
wants to understand XCP properly before writing a slave. If you're building
an XCP implementation from the spec (which I am), these are the things I
wish someone had underlined in red for me.

## The three-layer model, and why it's not negotiable

XCP is defined in three layers: **Application**, **Protocol**, and
**Transport**. The separation isn't academic. It's a hard constraint on how
you structure your code.

The Protocol Layer handles commands, DAQ, calibration, and memory access.
It speaks in *packets*: PID, optional counter, optional timestamp, data.
The Transport Layer wraps those packets for CAN, CAN FD, Ethernet, USB,
FlexRay, SxI. The packet content is identical regardless of transport.

What this means practically: if your Protocol Layer has any idea what
transport it's running on, you've broken the design. `CONNECT` doesn't know
about CAN IDs. `UPLOAD` doesn't know about USB endpoints. The transport
adapter is the only code that touches framing, and if you swap it out, the
entire rest of the stack keeps working. This is the whole reason ASAM drew
the boundary where they did.

The corollary is subtler. **Per-transport constants leak into the protocol
in exactly two places**, and you need to know where. `MAX_CTO` and
`MAX_DTO` (the maximum sizes for command and data packets) are
transport-dependent but get exposed to the Protocol Layer and negotiated
during `CONNECT`. Over CAN, you get 7 payload bytes per packet (the first
byte of the 8-byte CAN frame is the PID). Over Ethernet, you can send much
more. The master has to respect whatever the slave reports. Get this wrong
and your `DOWNLOAD` sequences silently truncate.

## CTO versus DTO: the split that makes everything work

XCP has two kinds of traffic: **Command Transfer Objects (CTO)** and **Data
Transfer Objects (DTO)**.

CTOs are command/response, synchronous, acknowledged. `CONNECT`, `UPLOAD`,
`SET_MTA`, `DOWNLOAD`: these are CTOs. The master sends, the slave
responds with `RES` (positive) or `ERR` (negative). Standard
request/response.

DTOs are the interesting ones. They carry measurement data (DAQ) and
stimulation data (STIM). They are **asynchronous, unacknowledged, and
event-driven from inside the ECU**. The slave fires them when an ECU event
fires, not when the master asks.

Why does this split matter? Because the naive approach to measurement
("master asks for variable X, slave replies with variable X's value") is
called *polling*, and the XCP spec explicitly positions it as the example
of what not to do for serious work. Polling has two fatal problems: it
doubles bus traffic (every read costs a request plus a response) and it
doesn't guarantee data correlation. If you poll `engine_rpm`, then
`throttle_pos`, then `lambda`, those three values came from three different
calculation cycles in the ECU. They don't correlate. They can't be plotted
meaningfully against each other.

DAQ fixes both problems. The master tells the slave once, up front, which
variables to sample and which ECU events to sample them on. Then it starts
the measurement and shuts up. The slave streams data as the events fire.
Every variable in a DAQ list is captured at the *same instant* (the event
itself), which is the only way to get values that correlate in time.

This split, synchronous commands vs. event-driven data streams, is the
single most important architectural fact about XCP. If your slave
implementation doesn't have a clean separation between command handling
(CTO) and event-driven packet emission (DTO), you've built it wrong.

## DAQ lists, ODTs, and the mental model that unlocks them

The DAQ configuration structure confuses people on first contact. Here's
the mental model that made it click for me.

An **event** is something that happens in the ECU at a known time: a 1 ms
task, a 10 ms task, an angle-synchronous event on the crank, a button
press. Events are defined by the ECU; the master reads them from the A2L
file.

A **DAQ list** is attached to exactly one event. When that event fires, the
DAQ list is transmitted. One event equals one DAQ list. If you want the same
variable measured at two different rates, you need two DAQ lists.

An **ODT (Object Descriptor Table)** is a frame's worth of payload
description. Over CAN, where you have 7 payload bytes per packet, one ODT
describes which variables go into those 7 bytes. If your DAQ list carries
20 bytes of measurement data over CAN, you need 3 ODTs in that DAQ list:
7 + 7 + 6 bytes.

So the hierarchy is: **Event, then DAQ List, then ODTs, then ODT entries
(address + length pointing into RAM)**.

The critical insight: the master is the one that constructs this layout
and sends it to the slave during configuration. The master decides which
variable goes where in which ODT. The slave doesn't choose, it just
obeys. This is why the master has to know the transport's `MAX_DTO`: it's
doing the packing.

### Static, predefined, dynamic: which one, when

Three flavours of DAQ lists exist, and the differences are operational:

- **Static**: the number of DAQ lists and ODTs is hard-coded in the ECU at
  build time. The variables in the ODTs are still master-selected, but the
  *shape* is fixed. Cheap and predictable, inflexible. If the user asks
  for more signals than fit, the slave rejects the config.
- **Predefined**: everything is frozen at build time, including the
  variables. Basically never used in ECU development. It exists for fixed
  analog measurement hardware.
- **Dynamic**: the slave reserves a memory pool, and the master allocates
  DAQ lists and ODTs out of it at runtime via `ALLOC_DAQ`, `ALLOC_ODT`,
  `ALLOC_ODT_ENTRY`. Flexible, more RAM-hungry, more complex slave code.
  This is what most ECU development uses.

A reasonable implementation order is static first, to get the protocol
state machine right, then dynamic allocation on top. The dynamic commands
are optional; the slave can reject them and the master has to cope.

## Timestamps: where the spec is generous and implementations are stingy

A DAQ packet can optionally carry a timestamp. "Optional" is doing a lot of
work in that sentence.

The timestamp is a free-running counter in the slave, incremented at a
known rate (documented in the A2L). When the slave emits a DAQ list, it
stamps the *first* ODT with the current counter value. The other ODTs in
the same list don't repeat the timestamp: they belong to the same event,
same instant. The master reconstructs timing from these stamps.

Three non-obvious points:

1. **The timestamp is the slave's clock, not the master's.** If your slave
   is a Cortex-M with a free-running 32-bit counter at 1 MHz, your
   timestamp resolution is 1 μs and it wraps every ~71 minutes. That wrap
   behaviour has to be handled on the master side.
2. **Multiple slaves on the same bus have independent clocks.** XCP v1.3+
   introduced `GET_DAQ_CLOCK_MULTICAST` and PTP-based synchronisation
   specifically to let the master correlate timestamps across slaves. For
   a single-slave setup, you can ignore all of this. For an instrumented
   car with multiple XCP slaves, you can't.
3. **Packed Mode changes the rules.** From XCP v1.4, Packed Mode lets the
   slave put multiple samples of the same signal into one ODT (for
   example, five consecutive values from a 1 MHz ADC) with only one
   timestamp for the group. The master reconstructs individual sample
   times via a fixed offset formula declared in the A2L. This is how you
   get high-rate measurement without drowning the bus in timestamps.

## The calibration trap: RAM vs Flash

XCP lets you change parameters in the ECU at runtime. Obvious. But the
question "where does the parameter live?" is where implementations
diverge, and where most of the real complexity hides.

Flash is read-any-byte, write-a-whole-block. You can't calibrate flash
contents directly at runtime, not in any meaningful sense. So the ECU has
to arrange for *tunable* parameters to live in RAM, either permanently or
on demand. This is what "calibration concept" means in the spec: the
strategy the ECU uses to make flash-stored initial values accessible for
RAM-based modification.

The four concepts from the spec, in order of increasing cleverness:

- **Parameters in RAM**: copy from flash to RAM at boot, application
  reads from RAM. Simple, but costs RAM permanently for every tunable.
- **Flash overlay**: hardware MMU or dedicated overlay feature maps a
  RAM region on top of a flash region. The application code references
  flash addresses; reads transparently hit RAM when the overlay is on.
  This is what lets you have "page switching": flip a bit and the whole
  calibration set swaps between flash values and tuned values
  instantaneously. Elegant, MCU-dependent.
- **Dynamic flash overlay allocation**: overlay RAM only for parameters
  actually being tuned right now. Solves the "not enough RAM for all
  tunables" problem at the cost of more complex slave logic.
- **Pointer-based (AUTOSAR)**: the application always dereferences
  through a pointer table. Changing a pointer swaps from flash to RAM for
  that parameter. The double-pointer variant lets you do page switching
  without touching every individual pointer.

Each of these has a distinct RAM cost, a distinct switching latency, and a
distinct failure mode when RAM runs out. The choice is rarely "which one
is best" and usually "which one does this MCU family actually support
cleanly". "Parameters in RAM" with a dedicated calibration segment is the
safe default when hardware overlay isn't available. Pointer-based is what
you inherit if you're working inside an AUTOSAR stack. Hardware overlay
with page switching is the nicest to use and the most MCU-dependent to
implement.

## The quiet gotchas

A few more things that cost me time when I dug into the spec:

- **Address extension is real and often ignored.** XCP addresses are
  40-bit: a 32-bit address plus an 8-bit extension. The extension lets
  the slave disambiguate memory spaces (internal vs external RAM,
  different CPU cores, etc.). A lot of tutorial code hard-codes extension
  to zero and loses portability later.
- **`SET_MTA` is stateful.** The Memory Transfer Address persists across
  commands. `UPLOAD` and `DOWNLOAD` increment it as they go. If you're
  writing a master and you interleave unrelated commands between setting
  MTA and using it, you'll get silent corruption.
- **`SHORT_UPLOAD` exists for a reason.** It carries the address inline,
  so you don't need a separate `SET_MTA`. Use it for one-shot reads. Use
  MTA plus `UPLOAD` only when you're streaming a block.
- **Block transfer mode has asymmetric rules.** The master has to respect
  the slave's `MIN_ST` (minimum separation time) and `MAX_BS` (max block
  size). The slave doesn't have to respect anything symmetric on the
  master side. The spec assumes the master is always fast enough. If
  you're writing a slow master, you'll find out the hard way.
- **`CONNECT` is not optional and neither is `GET_STATUS`.** Both must be
  implemented. Everything else in the Standard command group is optional,
  but those two are the floor.

---

*References: ASAM MCD-1 XCP v1.5 (Part 1 Overview, Part 2 Protocol Layer,
Part 3 Transport Layer) is the authoritative source for everything above.
Vector's "XCP - The Standard Protocol for Measurement and Calibration"
covers the same ground with more context and worked examples.*
