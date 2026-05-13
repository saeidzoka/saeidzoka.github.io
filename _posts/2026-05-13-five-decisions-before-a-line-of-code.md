---
layout: post
title: "Five Decisions Before a Line of Code: Designing an SxI Parser"
date: 2026-05-13
categories: xcp parser embedded design firmware
---

There's a particular kind of code I want to avoid writing. You've seen it.
A state machine that grew organically, where every bug fix added another
flag, every edge case added another branch, and the receive path now has
seven booleans whose interactions only the original author understood. It
works, usually. It also resists every attempt to add a feature without
breaking something three layers away.

The defence against that code isn't talent. It's deciding things before
you type. A protocol parser in particular has maybe five or six
architectural questions that, once settled, make the implementation almost
boring. Get them wrong and you spend the next month patching symptoms.

This is a working log of those five decisions for `xcp_transport_sxi`,
the SxI framing layer in my open-source XCP slave for the Raspberry Pi
Pico 2. The decisions are concrete, the tradeoffs are honest, and the
result is a parser that fit in one file and passed an automated test
suite on the first hardware run after a single targeted fix.

## 1. Linear buffer or ring buffer?

The reflex is "ring buffer, obviously". Ring buffers are the default
data structure for byte streams in embedded code. They handle producer
and consumer running at different rates, they absorb bursts, they look
sophisticated in code review.

For XCP, they are wrong.

XCP is a synchronous request-response protocol. The master sends one
command, waits for one response, then sends the next. There is never
more than one frame in flight in either direction. A ring buffer's
whole purpose is buffering *multiple* in-flight units while the
consumer catches up. We don't have multiple units. We have one.

A linear buffer sized to the maximum frame is enough:

```c
static uint8_t rx_buffer[XCP_SXI_MAX_FRAME];
static size_t  rx_index;
```

That's it. Bytes accumulate at `rx_index`, the frame completes, the
consumer takes it, `rx_index` returns to zero. The state machine knows
exactly where it is from the index alone.

Complexity has to earn its place. A ring buffer is more general, but
generality is a cost unless the use case demands it. The protocol
semantics here actively forbid the situation a ring buffer exists to
handle. Picking it because "that's what embedded code does" makes the
codebase worse, not safer.

## 2. How do you recover from going out of sync?

This question only matters if you accept that you *will* go out of sync.
Most parser bugs come from designers who quietly assumed they wouldn't.

The SxI frame is `[LEN(2)][CTR(2)][payload(LEN bytes)]`. No
start-of-frame marker, no checksum, no escape character. If the parser
shifts by one byte, the next two bytes it reads as `LEN` will produce a
nonsense length, the payload count will be wrong, and every frame after
that is corrupted.

USB CDC is a reliable transport. Bytes don't get flipped in transit and
they don't disappear. So how does out-of-sync happen at all? Two ways
that matter in practice:

1. The master sends a frame, gets halfway through, and disconnects (a
   crash, a cable yank, a host process killed mid-write).
2. The master sends a malformed `LEN` field. We can argue about whose
   bug that is, but the slave has to survive it.

Both of these leave us with bytes in `rx_buffer` that don't belong to
any complete frame, and a `rx_index` value that's lying about what state
we're in.

The recovery has two layers.

**Layer one is validation.** When the four-byte header is complete, the
parser checks `LEN <= XCP_MAX_CTO`. A value of 1000 isn't corruption on
USB CDC; it's a synchronisation error. The parser flushes the buffer,
increments `framing_errors`, and resets to `WAIT_HEADER`.

**Layer two is the inter-byte idle timeout.** Even after a flush, what
if the next byte isn't the start of a real frame? What if it's a
trailing payload byte from a frame the master abandoned, and that byte
happens to look like a legal `LEN` value? Without something else, the
parser silently resyncs to a phantom frame and corrupts everything
downstream.

The fix is to assume: *if the parser is mid-frame and nothing has
arrived for fifty milliseconds, the wire is dead and the buffer is
garbage*. Reset unconditionally. Fifty milliseconds is far longer than
any legitimate inter-byte gap on USB CDC, and far shorter than a
master's command timeout. It's the right backstop precisely because it
catches the case validation can't.

```c
if (rx_state != SXI_STATE_WAIT_HEADER || rx_index != 0) {
    if ((time_us_32() - last_rx_time_us) > IDLE_TIMEOUT_US) {
        reset_parser();
    }
}
```

That's the whole recovery strategy: validate when you can, time out
when you can't.

## 3. Polling the clock, not interrupting on it

Once you accept that the parser needs a timeout, the next decision is
how to detect it. The reflex this time is "set up a hardware timer,
have it fire an interrupt, run the recovery in the ISR". Timers are
free on every modern MCU. Interrupts are responsive.

This is also wrong, for less obvious reasons.

An interrupt-driven timeout means the recovery code runs in interrupt
context. That code touches the same state machine variables the task
function touches. Suddenly you need atomic access to `rx_state` and
`rx_index`. You need to disable interrupts around the parser's read
of those variables. The state machine, which was a clean cooperative
piece of code, has to defend itself against pre-emption.

The cost of that defence is paid in every read and every write to the
parser's state, forever, for a check that fires once every few minutes
under abnormal conditions. The cost of *not* having interrupts is one
extra subtraction inside `xcp_transport_sxi_task()`:

```c
uint32_t now = time_us_32();
if ((now - last_rx_time_us) > IDLE_TIMEOUT_US) {
    /* recovery */
}
```

The task is already called every iteration of the main loop. The timer
check is one comparison. There's no race condition because there's no
concurrent access. The cooperative model that the rest of the firmware
relies on stays intact.

Interrupts are a tool for problems the main loop can't service in time.
A fifty millisecond timeout serviced by a main loop that runs in
microseconds is not such a problem. The reflex toward interrupts here
buys nothing and costs the cooperative invariants that make the rest
of the code easy to reason about.

One implementation detail worth keeping: the comparison uses unsigned
subtraction. `time_us_32()` is a thirty-two bit microsecond counter and
wraps every seventy-one minutes. Unsigned subtraction handles the wrap
transparently as long as the timeout is shorter than half the counter
range. Computing `now - last_rx_time_us` and comparing against
`IDLE_TIMEOUT_US` is correct regardless of where the counter is in its
cycle. Computing `now > last_rx_time_us + IDLE_TIMEOUT_US` is not. The
difference is invisible until the counter wraps mid-frame in production
and the parser locks up for no reason anyone can reproduce.

## 4. Three states, not four

The SxI frame has two header fields, `LEN` and `CTR`, each two bytes.
The naive state machine has four states: `WAIT_LEN_LOW`, `WAIT_LEN_HIGH`,
`WAIT_CTR_LOW`, `WAIT_CTR_HIGH`, then `WAIT_PAYLOAD`. Each byte
transitions the state forward.

This is wrong, for a reason that matters beyond this parser.

The header is an atomic unit. Until all four bytes have arrived, the
parser cannot make any decision. `LEN` alone is meaningless without
`CTR`. There is no useful intermediate state where we have `LEN` but
not `CTR`, because we don't act on `LEN` until we have everything.

Compressing the four states into one (`WAIT_HEADER`) doesn't lose
information; it accurately represents what's happening:

```c
typedef enum {
    SXI_STATE_WAIT_HEADER,   /* accumulating bytes 0..3   */
    SXI_STATE_WAIT_PAYLOAD,  /* accumulating bytes 4..N-1 */
    SXI_STATE_FRAME_READY,   /* consumer must drain       */
} sxi_rx_state_t;
```

Three states, three transitions. The parser reads bytes into
`rx_buffer` until `rx_index` reaches four, parses the header in one
shot, transitions to `WAIT_PAYLOAD` (or back to `WAIT_HEADER` on
validation failure), continues until `rx_index` reaches
`SXI_HEADER_SIZE + LEN`, transitions to `FRAME_READY`. The consumer
drains, the parser resets, the cycle repeats.

The `FRAME_READY` state earns its place because of *back-pressure*.
While a frame is waiting for the consumer, the parser refuses to read
more bytes from the USB layer. The frame in `rx_buffer` is sacred until
collected. New bytes accumulate in the lower transport's RX FIFO, which
is fine: that's what FIFOs are for.

The general principle: *states should represent decision points, not
data accumulation points*. Bytes accumulate inside a state; transitions
happen when something becomes knowable. If you don't act on a byte's
arrival, the byte doesn't deserve its own state.

## 5. Switch dispatch beats a function pointer table here

The last decision is the structure of the dispatch itself. Two
patterns are common in embedded state machines:

**Switch-driven.** A `switch` on the state variable, one case per
state, the handling inline.

**Table-driven.** An array of function pointers indexed by state. The
dispatch is `handlers[state](byte)`.

Table-driven dispatch is genuinely beautiful in the right place. In
AUTOSAR's diagnostic stack (DCM), where there are dozens of UDS
services each handled by a homogeneous `(buffer, length) -> response`
function, the table approach is exactly right. New services slot in
without changing the dispatch code. Code generation can build the
table from XML. The pattern earns its overhead because it scales.

It does not scale down. With three states, all of which do different
things and need different parameters, a table buys nothing and costs
several things. Function pointers cost an indirect call (which Cortex-M
predicts worse than direct calls). They cost a `noreturn` analysis path
that confuses static analysers. They cost the discipline of forcing all
handlers into a single signature even when the natural signatures
differ. MISRA-C 2012 Rule 18.4 raises function pointers as something
that needs justification, and *justification needed* is the wrong
default for code with three states.

A switch with three cases is two comparisons and a jump on most ARM
compilers, which emit a literal jump table for dense enums anyway. The
performance is identical to a hand-written table, the code is shorter,
and the analyser is happier.

The deciding question isn't "which pattern is more advanced". It's
*does the state space want to grow*? In `xcp_transport_sxi`, the state
space is defined by a protocol that hasn't changed in two decades. It
will not grow. So the structure that's optimised for non-growth is the
right one.

## What you take away

Five decisions, fifteen minutes each, an afternoon's worth of design
work before any code existed. The implementation that followed was
roughly two hundred lines of C, took a couple of hours to write, and
needed exactly one bug fix after the automated test suite caught a
USB CDC FIFO interaction that wasn't visible in manual testing.

The fix itself was small. It would have been larger, and might not have
worked at all, if the underlying state machine and recovery strategy
had been wrong. Good design doesn't prevent bugs. It prevents the kind
of bugs that survive contact with hardware because the architecture
couldn't accommodate the truth that arrived from the wire.

The pattern that runs through all five of these decisions is the same.
*Match the structure to the problem you actually have*, not to the
problem the textbook describes. Ring buffers, interrupts, function
pointer tables, fine-grained states: all of these are excellent tools
when the problem calls for them, and dead weight when it doesn't. The
work before the work is figuring out which.

The full source for the parser lives at
[github.com/saeidzoka/xcp-pico](https://github.com/saeidzoka/xcp-pico),
specifically in `firmware/src/xcp_transport_sxi.c`. The commit history
walks through these decisions in order: interface definition, scaffold,
state machine, frame I/O, integration, then the one bug fix the design
made small enough to handle in a single targeted change. Open it, read
the commits, see whether the reasoning here matches the code there.

---

*References: ASAM MCD-1 XCP v1.5 Part 2 (SxI Transport Layer). The
project's ADR-001 documents the decision to carry SxI framing over
USB CDC rather than using XCP-on-USB directly.*
