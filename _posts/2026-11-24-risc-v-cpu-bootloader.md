---
layout: post
title: "A bootloader for my RISC-V CPU: from eight minutes to one second"
date: 2026-11-24
tags: [riscv, verilog, fpga, ai]
---

[Last time]({% post_url 2026-11-23-risc-v-cpu-can-listen %}), my RISC-V core, [**rv-ai-lab**](https://github.com/giuseros/rv-ai-lab), learned to receive characters over a serial line. That was mostly a means to an end. Until now, the program the CPU runs was the RAM's initial contents, and those are baked into the FPGA image. Change one line of C, and Quartus spends eight minutes building a new image.

Now it takes a second:

```
$ make load PROG=primes
Press KEY[0] on the board (reset)...
sending primes.bin: 6800 bytes, CRC a7b5cbb6, 0.6 s
boot: ok after 0.6 s; running it. Ctrl+] quits.
primes below 1000:
   2   3   5   7  11  13  17  19  23  29
  31  37  41  43  47  53  59  61  67  71
...
```

## A bootloader is just a program

There's nothing special about a bootloader. It's the program that's in memory when the board starts, and its only job is to get another program and run it. Mine waits on the serial line, stores what arrives in RAM, checks it, and jumps to it. It's about 1 KB of C.

The FPGA image now holds the CPU and the bootloader, and it no longer changes when the program does. I rebuild it only when I change the hardware.

## Two programs in one memory

After reset, the CPU always starts at address 0. So the bootloader lives at 0 and has the first 2 KiB to itself: its code, its data and its stack. Programs go above it, from address `0x800`.

<figure style="margin: 1.5em 0;">
<svg viewBox="0 0 720 320" width="100%" role="img" aria-label="Left: the 16 KiB RAM. From 0x0000 to 0x0800 the bootloader, with its code, data and stack. From 0x0800 the program it received: its code and initialised data, then its zeroed globals, then free memory where the program's stack grows down from 0x4000. Right: what the laptop sends: the 4 bytes RVLD, a 4-byte length, the program bytes, which are stored from 0x0800, and a 4-byte CRC-32. The bootloader answers boot: ok and jumps to 0x800, or reports bad length, bad checksum or bytes lost and waits for the next frame." xmlns="http://www.w3.org/2000/svg" style="font-family: inherit; font-size: 13px;">
<style>
.bl-box { fill: var(--bg-card, #fff); stroke: var(--text-muted, #6b7280); stroke-width: 1.2; }
.bl-new { fill: var(--accent-soft, #eef0fe); stroke: var(--accent, #4f46e5); stroke-width: 1.4; }
.bl-grp { fill: var(--bg-subtle, #f7f8fa); stroke: var(--text-muted, #6b7280); stroke-width: 1.2; }
.bl-t { fill: var(--text, #1a1d23); }
.bl-b { fill: var(--text, #1a1d23); font-weight: 600; }
.bl-m { fill: var(--text-muted, #6b7280); font-size: 11.5px; }
.bl-mono { font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; font-size: 11px; }
.bl-line { stroke: var(--text-muted, #6b7280); stroke-width: 1.4; fill: none; }
.bl-head { fill: var(--text-muted, #6b7280); }
</style>
<defs>
<marker id="bl-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="bl-head"/></marker>
</defs>
<!-- the RAM -->
<text class="bl-b" x="175" y="28" text-anchor="middle">RAM · 16 KiB</text>
<rect class="bl-box" x="90" y="40" width="170" height="70"/>
<text class="bl-b" x="175" y="70" text-anchor="middle">bootloader</text>
<text class="bl-m" x="175" y="87" text-anchor="middle">code, data, its stack</text>
<rect class="bl-new" x="90" y="110" width="170" height="80"/>
<text class="bl-b" x="175" y="146" text-anchor="middle">the program</text>
<text class="bl-m" x="175" y="163" text-anchor="middle">code + initialised data</text>
<rect class="bl-box" x="90" y="190" width="170" height="30"/>
<text class="bl-m" x="175" y="209" text-anchor="middle">its zeroed globals (.bss)</text>
<rect class="bl-grp" x="90" y="220" width="170" height="80"/>
<text class="bl-t" x="175" y="252" text-anchor="middle">free</text>
<text class="bl-m" x="175" y="269" text-anchor="middle">its stack grows down</text>
<text class="bl-t bl-mono" x="82" y="45" text-anchor="end">0x0000</text>
<text class="bl-t bl-mono" x="82" y="115" text-anchor="end">0x0800</text>
<text class="bl-t bl-mono" x="82" y="300" text-anchor="end">0x4000</text>
<!-- the frame -->
<text class="bl-b" x="300" y="62">what make load sends</text>
<rect class="bl-box" x="300" y="75" width="70" height="36" rx="4"/>
<text class="bl-t bl-mono" x="335" y="97" text-anchor="middle">RVLD</text>
<rect class="bl-box" x="372" y="75" width="70" height="36" rx="4"/>
<text class="bl-t" x="407" y="98" text-anchor="middle">length</text>
<rect class="bl-new" x="444" y="75" width="170" height="36" rx="4"/>
<text class="bl-t" x="529" y="98" text-anchor="middle">the program</text>
<rect class="bl-box" x="616" y="75" width="80" height="36" rx="4"/>
<text class="bl-t" x="656" y="98" text-anchor="middle">CRC-32</text>
<text class="bl-m" x="335" y="128" text-anchor="middle">magic</text>
<text class="bl-m" x="407" y="128" text-anchor="middle">4 bytes</text>
<text class="bl-m" x="529" y="128" text-anchor="middle">length bytes</text>
<text class="bl-m" x="656" y="128" text-anchor="middle">4 bytes</text>
<path class="bl-line" d="M529,136 C529,180 390,150 264,150" marker-end="url(#bl-arrow)"/>
<text class="bl-m" x="455" y="182" text-anchor="middle">stored from 0x800</text>
<!-- the answers -->
<text class="bl-b" x="300" y="226">then one line back</text>
<text class="bl-t bl-mono" x="300" y="248">boot: ok</text>
<text class="bl-m" x="370" y="248">jump to 0x800: the program runs</text>
<text class="bl-t bl-mono" x="300" y="272">boot: bad length / bad checksum / bytes lost</text>
<text class="bl-m" x="300" y="290">nothing runs; it waits for the next frame</text>
</svg>
<figcaption style="text-align: center; font-size: .9em; color: var(--text-muted);">Where everything lives in RAM, and what the laptop sends to put a program there.</figcaption>
</figure>

That means a program has to be built to run at `0x800`. Compiled code is full of absolute addresses: where its globals are, the targets of its jump tables, its function pointers. Copy a program built for address 0 up to `0x800` and all of those still point into the bootloader. So the linker script now takes the start address as a parameter, and `make load` links every program for `0x800`.

My test for this is the C self-test from an earlier step, which uses jump tables, function pointers and initialised globals. It only passes if every one of those addresses is right, so it only passes if the bytes landed intact, exactly where they were linked to run.

## The frame

The laptop sends a short header, the program, and a checksum, as in the picture.

The bootloader doesn't know when a program is coming. It just sees bytes. So it keeps the last four bytes it received and waits until they spell `RVLD`. Whatever came before is skipped, whether it's me typing or half a frame from a send I interrupted.

The CRC at the end tells it whether the bytes it stored are the bytes that were sent. A bad cable can flip a bit. A byte can arrive while the receive buffer is full, or with a broken stop bit, and the receiver flags both of those. In every case the bootloader answers with what went wrong, and nothing runs.

There are now three separate implementations of this CRC:
- the bootloader's, with a 16-entry table, because 64 bytes of table fit in a bootloader
- the testbench's, one bit at a time, straight from the definition
- Python's `zlib` on the laptop

All three agree, which is good evidence that each of them is right.

## The wire is the bottleneck

At 115200 baud, a byte arrives every 2,170 cycles of the 25 MHz clock. Storing it and adding it to the CRC takes a few dozen. So the CPU spends almost all its time waiting, and a 6.8 KB program loads in 0.6 s. That's simply how fast the wire is.

## What should reset do?

The first version surprised me. I loaded a program, it ran, then I pressed reset (KEY[0]) and expected it to run again. Instead I got `boot: ready`: the board was waiting for a new program.

Reset doesn't erase memory, so the program was still there. But reset sends the CPU to address 0, and address 0 is the bootloader.

So now it behaves like an Arduino. After reset it waits about a second for a new program, and if none comes, it reruns the last one. There's no timer yet, so "a second" is a loop that runs a fixed number of times, about a second at 25 MHz.

This turned out to be subtler than it sounds, because some programs can't just be run again. Take a global like this:

```c
int counter = 5;
```

No instruction ever stores that 5. It's in the program image, and loading the image is what puts it in RAM. If the program changes `counter`, nothing ever puts the 5 back. A microcontroller with flash memory has the same problem and solves it with a copy: its startup code copies the initial values from flash into RAM at every boot. Here, the RAM is the only copy.

So the bootloader keeps a small record of the last program it loaded, with its length and CRC, in a corner of its own memory that reset doesn't touch. Before a rerun, it recomputes the CRC of the program in RAM:
- **If it matches**, the program is untouched, and it runs again.
- **If it doesn't**, the bootloader says `boot: last program changed, send it again`.

Globals without an initial value are fine. They aren't part of the image, and the program's startup code zeroes them every time it starts.

The record starts with a magic number, so that whatever the memory held at power-on is never mistaken for a record. One test fills memory with a pattern whose fake length looks plausible. A version of the bootloader without the magic check fails it.

## A check that could never fail

One last lesson. The linker script checks that a program leaves enough room for its stack. When I added the rerun, the bootloader outgrew the 1 KiB it had at first, and the build still passed.

The check was written as a subtraction of unsigned numbers: top of the stack minus end of the program. Once the program ran past the top, the subtraction wrapped around to a huge number, and "is there room?" said yes. It's now written as a sum, which can't wrap, and programs moved up from `0x400` to `0x800`.

It was the same lesson as before: a check needs a test that proves it can fail.

## What's next

Trying a program on the board is now as quick as running it in the simulator, so I can measure the design on real hardware instead of guessing. For that the CPU needs to measure itself, with counters for cycles and instructions. After that comes the big one: pipelining the core.

The code, tests and all, is on GitHub: [**giuseros/rv-ai-lab**](https://github.com/giuseros/rv-ai-lab).
