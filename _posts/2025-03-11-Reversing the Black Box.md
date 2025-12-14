---
layout: post
title:  "Reversing the Black Box: Running TriCore Assembly on Bare Metal"
date:   2025-03-11 12:10:00 +0330
categories: automotive
---

If you’ve ever tried to get into automotive reverse engineering, you’ve likely hit the "hardware wall." To even say "Hello" to an Infineon TriCore chip (the brain inside most modern ECUs) you usually need a debugger that costs as much as a used car.

But recently, I decided to challenge that norm. I wanted to see if I could write, compile, and debug raw TriCore assembly completely for free, using nothing but open-source tools. My goal wasn't just to blink a virtual LED; I wanted to implement a real-world **Seed-to-Key algorithm**, the kind used for security access in ECUs, and run it on QEMU.

It turned out to be a journey full of undocumented quirks, syntax wars, and one very annoying GDB crash. Here is how it went down.

### The Mission: Porting Logic to Assembly

I started with a classic C function for generating a security key. The logic is simple: take a seed, loop 35 times, check the MSB (Most Significant Bit), shift, and conditionally XOR with a mask.
```
uint32 Gp_SeedTokey(uint32 seed,uint32 mask)
{
    uint8 i;
    uint32 key = 0;
    if(seed != 0)
    {
        for(i = 0; i < 35; i++)
        {
            if(seed & 0x80000000)
            {
                seed= seed <<1;
                seed = seed ^ mask;
            }
            else
            {
                seed= seed <<1;
            }
        }

        key = seed;
    }

    return key;
}
```

In C, this is trivial. In TriCore assembly, however, you immediately face the reality of a split-register architecture. Unlike x86 or ARM where registers are mostly general-purpose, TriCore strictly separates **Data (`%d`)** and **Address (`%a`)** registers. You can't do math on an address register, and you can't dereference a data register.

I set up my environment using the HighTec GCC toolchain and QEMU. My first hurdle wasn't the logic, but the assembler itself.

### The "GNU Syntax" Trap

If you read the official Infineon architecture manual, you’ll see instructions like `mov d0, #100`. But if you feed that to the GNU assembler (`tricore-as`), it screams at you.

It turns out, the open-source tools demand a different dialect. You have to prefix registers with `%`, and crucially you must **not** use the `#` sign for immediate values. In the GNU world, `#` starts a comment. So, my innocent instruction `mov %d0, #100` was being interpreted as "Move d0," followed by a comment, leading to confusing syntax errors.

Once I cleaned up the syntax, I faced the hardware constraints.

### The 32-Bit Immediate Problem

The Seed-to-Key algorithm checks the MSB using a mask: `0x80000000`. In my naive first draft, I wrote: `and %d7, %d7, 0x80000000`

The assembler rejected this immediately. Why? Because TriCore instructions generally encode immediate values in small chunks (often 9 bits). You cannot jam a full 32-bit integer into a standard `AND` instruction.

To solve this, I had to construct the value manually using the `movh` (Move High) instruction, which loads data into the upper 16 bits of a register:

```
movh %d8, 0x8000      /* Loads 0x8000 into the upper half -> 0x80000000 */
and  %d7, %d7, %d8    /* Now we can AND register to register */

```

This is the kind of low-level optimization you miss when writing in C, where the compiler silently handles these register gymnastics for you.

### The Missing "JNZ"

Another surprise was the flow control. Coming from x86, I instinctively tried to use `jnz` (Jump if Not Zero). It doesn't exist. TriCore forces you to be explicit. You have to use `jne` (Jump if Not Equal) and compare your register against a literal zero.

After rewriting the loop and handling the memory pointers (using TriCore's slick post-increment addressing `[%a4+]4`), I finally had a binary ready for the virtual hardware.

### Debugging the Ghost Machine

Running the code is where things got interesting. I used QEMU to emulate a generic TriCore board (`tricore_testboard`) and attached GDB to inspect the memory.

`qemu-system-tricore -M tricore_testboard -kernel seed_key.elf -nographic -s -S`

I fired up GDB, connected to localhost, and decided to use the Text User Interface (`layout regs`) to watch my registers update in real-time.

**Bad idea.**

The screen garbled, and GDB threw a `Could not fetch register "lcx"` error. It turns out, the current QEMU/GDB combination has a bug where it fails to retrieve specific context management registers, causing the UI to crash loop.

The fix? Abandoning the fancy UI. I had to go old-school, stepping through instructions one by one with `si` and manually inspecting registers with `info registers`.

### The Moment of Truth

Despite the crashes and syntax errors, I eventually stepped through the loop. I watched the `input_seed_mem` load from address `0x80000000`, saw the XOR operations toggle the bits in register `%d4`, and finally, the calculated key landed in `%d2`.

Here is the final, working assembly snippet that survived the process:

```
/* Part of the SeedToKey Loop */
loop_start:
    ge      %d15, %d6, 35        /* Check loop counter */
    jne     %d15, 0, loop_end    /* Branch if done */

    mov     %d7, %d4
    and     %d7, %d7, %d8        /* Check MSB using our constructed mask */
    sh      %d4, %d4, 1          /* Shift left */
    
    jeq     %d7, 0, skip_xor     /* If MSB was 0, skip the XOR */
    xor     %d4, %d4, %d5        /* The magic XOR */
    
skip_xor:
    add     %d6, 1
    j       loop_start

```

### Why Bother?

You might ask why anyone would hand-write assembly for a chip that has excellent C compilers. The answer isn't efficiency, it's understanding.

When you are reverse engineering an ECU dump, you don't get C source code. You get this: raw instructions, weird addressing modes, and compiler optimizations. By struggling through the process of writing it yourself, you learn to recognize the patterns. You understand why a decompiler output looks the way it does.

And honestly? There is a unique satisfaction in seeing those registers flip to the correct values, knowing you are controlling the silicon directly, even if that silicon is just a QEMU process on a Linux terminal.
