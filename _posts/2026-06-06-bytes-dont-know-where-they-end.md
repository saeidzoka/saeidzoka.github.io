---
layout: post
title: "Bytes Don't Know Where They End"
date: 2026-06-06
categories: [embedded, protocols]
---

Send a hundred bytes over UART and ask your receiver how many packets
arrived. It cannot answer. UART gives you bytes. It has no concept of a
packet. This distinction matters more than it seems, and it fails in a
specific, predictable way.

## Two Kinds of Transports

When a CAN frame arrives, you get a `dlc` field and a data buffer. The frame
boundary is part of the protocol. Ethernet works the same way: a frame has a
defined structure, a length, and a checksum. You receive a complete frame or
you receive nothing.

UART, SPI in byte mode, USB CDC: you get a byte stream. There is no frame
boundary. There is no length field. Bytes arrive one at a time, and the
transport has no opinion about what they mean or where one message ends and
the next begins.

On packet transports, framing is solved for you. On byte stream transports,
it is your problem.

## The Failure You Will Not See in Testing

A naive receiver reads a fixed number of bytes, processes them as a command,
and waits for the next batch. This works in a lab: clean cable, short bursts,
no interruptions.

In the field, it fails. A cable interruption mid-frame leaves the receiver
waiting for bytes that will never arrive. The next valid transmission from the
sender gets merged with the tail of the partial frame. The receiver processes
garbage. No error is reported. The session appears to be functioning. The
failure is silent and deferred.

Recovery requires a reset, a timeout, or a sentinel byte that signals a new
frame start. None of these happen automatically.

## Adding What the Transport Forgot

The direct fix is a length prefix. Before each payload, send the number of
bytes that follow:

```
[LEN : 2 bytes][PAYLOAD : LEN bytes]
```

The receiver reads two bytes, learns how much to expect, reads exactly that
many, then processes the packet. If LEN bytes do not arrive within some
timeout, the receiver flushes its buffer and starts over. Sync is recoverable
without a reset.

ASAM's SxI format adds one field to this:

```
[LEN : 2 bytes][CTR : 2 bytes][PAYLOAD : LEN bytes]
```

`CTR` is a rolling counter incremented by the sender with each packet. The
receiver checks whether CTR advanced by exactly one from the previous frame.
If it jumped by more, a packet was dropped somewhere. The receiver logs the
gap and accepts the frame anyway.

Detecting loss is not the same as recovering from it, and recovery belongs to
the layer above. The framing layer reports what happened. The protocol layer
decides what to do about it. This is a clean separation that is easy to
violate and hard to undo once violated.

## Losing Sync and Getting It Back

A length-prefix parser has one vulnerability. If it loses sync, it reads
arbitrary bytes into the LEN field and then waits for that many bytes to
arrive. If those bytes happen to encode a large number, the parser stalls
waiting for data that will never come.

The fix is an idle timeout. The parser tracks the time since the last byte
arrived. If that interval exceeds a threshold, the buffer is flushed and the
parser resets to its initial state. The next byte from the sender begins a
fresh frame.

The threshold is a judgment call. In xcp-pico's SxI implementation it is
50 milliseconds, chosen to sit comfortably above USB CDC's variable
inter-byte latency while staying short enough to recover quickly after a
disruption. For a calibration protocol with human-scale interaction, that
window is wide enough on both sides.

One implementation note worth making: this timeout is polled in the main loop
rather than driven by a hardware timer interrupt. XCP is a synchronous
request-response protocol. Only one frame is in flight at a time, so the main
loop always reaches the timeout check before the next frame could arrive. A
ring buffer, interrupt-driven approach would be appropriate for a streaming
protocol with high throughput; for XCP, it would be complexity without
benefit.

## When the Transport Already Did the Work

CAN DLC is your length field. A CAN frame arrives complete or not at all. You
read `data[0..dlc-1]` and you have a packet. There is nothing to reconstruct.

This is why XCP on CAN carries no SxI header. The transport provides what SxI
provides: a length, a packet boundary, and error detection. Adding a framing
layer on top would be redundant.

It is also why the ASAM specification notes, without elaboration, that "XCP on
USB has no practical significance." The formal XCP-on-USB transport defined in
Part 3 uses USB bulk endpoints with device-class semantics that give you
packet boundaries at the endpoint level. USB CDC, the virtual serial port, is
a different thing: it presents as a byte stream. So a receiver using USB CDC
faces the same framing problem as a receiver on UART. SxI solves it in both
cases.

The sentence in the spec is about the complexity of implementing a correct USB
device class for calibration workflows, not about the wire itself. The wire
works fine.

## The State Machine

Once you have decided on a framing format, the receiver is a straightforward
state machine. For length-prefix-plus-counter, three states are sufficient:

```
WAIT_HEADER     Accumulate bytes until LEN and CTR are complete.
WAIT_PAYLOAD    Read exactly LEN more bytes.
FRAME_READY     A complete packet is in the buffer.
```

FRAME_READY is a distinct state rather than an immediate dispatch because the
consumer might not be ready to process the packet yet. Separating reception
from processing gives the caller control over timing without forcing the
receiver into a polling loop on the consumer side.

In each state, the idle timeout runs. A timeout in any state resets to
WAIT_HEADER and discards whatever was accumulated. This is the only recovery
path needed for a reliable transport with low frame loss rates. If the
underlying transport were unreliable, you would need retransmission, sequence
numbers, and acknowledgements. USB CDC, like a wired UART, does not lose bytes
under normal conditions. The timeout handles the abnormal case.

---

The framing problem is not specific to XCP or to calibration protocols. Any
structured communication over a byte stream faces it. The choices are the same
every time: delimiter-based, length-prefix, or something like COBS. The
trade-offs are known. ASAM's SxI format makes a reasonable set of them, and
the state machine that implements it fits comfortably in a few hundred lines of
C. Worth understanding once.

---

## References

- ASAM MCD-1 XCP V1.5, Part 3: XCP on SxI Transport Layer Specification
- Vector XCP Reference Book V3.0, Section 1.4.5
- xcp-pico `xcp_transport_sxi` module:
  [github.com/saeidzoka/xcp-pico](https://github.com/saeidzoka/xcp-pico)
