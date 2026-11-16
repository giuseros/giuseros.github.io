---
layout: post
title: "My own RISC-V CPU is running C on an FPGA"
date: 2026-11-16
tags: [riscv, verilog, fpga, ai]
---

[Last time]({% post_url 2026-11-06-rv-ai-lab %}), my little RISC-V core, [**rv-ai-lab**](https://github.com/giuseros/rv-ai-lab), had just printed "hello" in a simulator, and the plan was to run C compiled with GCC. Ten days later it's running C, and not in a simulator: on a real FPGA, on my desk.

```
primes below 1000:
   2   3   5   7  11  13  17  19  23  29
  31  37  41  43  47  53  59  61  67  71
 ...
 947 953 967 971 977 983 991 997
168 primes, sum 76127
```

Here's what's going on:

<figure style="margin: 1.5em 0;">
<svg viewBox="0 0 680 290" width="100%" role="img" aria-label="Diagram: primes.c is compiled by GCC into a program image, which Quartus bakes into the FPGA bitstream. On the DE10-Lite board, the RISC-V CPU runs the program from RAM, prints through a UART to a USB-serial adapter and the laptop, and shows its exit code 01295F on the 7-segment displays." xmlns="http://www.w3.org/2000/svg" style="font-family: inherit; font-size: 13px;">
<style>
.rv-box { fill: var(--bg-card, #fff); stroke: var(--text-muted, #6b7280); stroke-width: 1.2; }
.rv-cpu { fill: var(--accent-soft, #eef0fe); stroke: var(--accent, #4f46e5); stroke-width: 1.6; }
.rv-board { fill: var(--bg-subtle, #f7f8fa); stroke: var(--accent, #4f46e5); stroke-width: 1.2; stroke-dasharray: 6 4; }
.rv-t { fill: var(--text, #1a1d23); }
.rv-b { fill: var(--text, #1a1d23); font-weight: 600; }
.rv-m { fill: var(--text-muted, #6b7280); font-size: 11.5px; }
.rv-mono { font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; }
.rv-seg { fill: var(--accent, #4f46e5); font-size: 24px; letter-spacing: 4px; font-weight: 600; }
.rv-line { stroke: var(--text-muted, #6b7280); stroke-width: 1.4; fill: none; }
.rv-head { fill: var(--text-muted, #6b7280); }
</style>
<defs>
<marker id="rv-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="rv-head"/></marker>
</defs>
<!-- source to bitstream -->
<rect class="rv-box" x="20" y="25" width="110" height="40" rx="6"/>
<text class="rv-t rv-mono" x="75" y="50" text-anchor="middle">primes.c</text>
<line class="rv-line" x1="130" y1="45" x2="198" y2="45" marker-end="url(#rv-arrow)"/>
<text class="rv-m" x="164" y="38" text-anchor="middle">GCC</text>
<rect class="rv-box" x="200" y="25" width="140" height="40" rx="6"/>
<text class="rv-t" x="270" y="50" text-anchor="middle">program image</text>
<!-- the board -->
<rect class="rv-board" x="20" y="100" width="420" height="178" rx="10"/>
<text class="rv-m" x="430" y="118" text-anchor="end">DE10-Lite · MAX 10 FPGA</text>
<!-- drawn after the board, so its fill does not cover the arrow -->
<path class="rv-line" d="M270,65 V82 H95 V127" marker-end="url(#rv-arrow)"/>
<text class="rv-m" x="282" y="86">Quartus bakes it into the FPGA bitstream</text>
<rect class="rv-box" x="40" y="130" width="110" height="50" rx="6"/>
<text class="rv-b" x="95" y="152" text-anchor="middle">RAM</text>
<text class="rv-m" x="95" y="168" text-anchor="middle">the program</text>
<line class="rv-line" x1="152" y1="155" x2="188" y2="155" marker-start="url(#rv-arrow)" marker-end="url(#rv-arrow)"/>
<rect class="rv-cpu" x="190" y="130" width="110" height="50" rx="6"/>
<text class="rv-b" x="245" y="152" text-anchor="middle">RISC-V CPU</text>
<text class="rv-m" x="245" y="168" text-anchor="middle">RV32I · 25 MHz</text>
<line class="rv-line" x1="300" y1="155" x2="328" y2="155" marker-end="url(#rv-arrow)"/>
<rect class="rv-box" x="330" y="130" width="90" height="50" rx="6"/>
<text class="rv-b" x="375" y="152" text-anchor="middle">UART</text>
<text class="rv-m" x="375" y="168" text-anchor="middle">115200 baud</text>
<line class="rv-line" x1="245" y1="180" x2="245" y2="213" marker-end="url(#rv-arrow)"/>
<text class="rv-m" x="252" y="201">exit code</text>
<rect class="rv-box" x="190" y="215" width="230" height="45" rx="6"/>
<text class="rv-seg rv-mono" x="305" y="246" text-anchor="middle">01295F</text>
<!-- off the board -->
<line class="rv-line" x1="420" y1="155" x2="463" y2="155" marker-end="url(#rv-arrow)"/>
<text class="rv-m" x="452" y="147" text-anchor="middle">pin</text>
<rect class="rv-box" x="465" y="130" width="85" height="50" rx="6"/>
<text class="rv-b" x="507" y="152" text-anchor="middle">USB-serial</text>
<text class="rv-m" x="507" y="168" text-anchor="middle">PL2303</text>
<line class="rv-line" x1="550" y1="155" x2="578" y2="155" marker-end="url(#rv-arrow)"/>
<rect class="rv-box" x="580" y="130" width="90" height="50" rx="6"/>
<text class="rv-b" x="625" y="152" text-anchor="middle">laptop</text>
<text class="rv-m rv-mono" x="625" y="168" text-anchor="middle">picocom</text>
<text class="rv-m rv-mono" x="625" y="200" text-anchor="middle">168 primes,</text>
<text class="rv-m rv-mono" x="625" y="214" text-anchor="middle">sum 76127</text>
</svg>
<figcaption style="text-align: center; font-size: .9em; color: var(--text-muted);">From C source to primes on the laptop. The dashed box is the board; the CPU, RAM and UART are Verilog running on its FPGA.</figcaption>
</figure>

- A little C program (a prime sieve) is compiled with GCC for RISC-V. There's no operating system and no C library: just a tiny startup file, a linker script and a home-made `printf`.
- The program is baked into the FPGA bitstream as the initial contents of the RAM.
- The CPU runs on a Terasic **DE10-Lite**, an Intel/Altera MAX 10 board.
- `printf` goes to a UART written in Verilog, out of a pin on the GPIO header, into a cheap PL2303 USB-serial adapter, and into `picocom` on my laptop.
- When `main` returns, its return value, the sum of the primes (76127 = 0x01295F), shows up on the board's six 7-segment displays: `01295F`.

The best part: the output from the board matched the simulator **byte for byte, on the first try**. That's the simulation-first rule paying off. Nothing went near the board before its testbench passed, and every new testbench got the mutation treatment from last time: break the hardware on purpose and make sure the test notices.

A few things I learned along the way:

- **FPGA memory answers a cycle late.** The simulator's memory answered instantly. Real block RAM gives you the data on the clock edge *after* you ask, so the CPU has to put out the address of the next instruction one cycle early.
- **Quartus can quietly throw away your RAM contents.** Written one way, it built the RAM but dropped the program in it, so the CPU would have woken up to an empty memory. Written another way, it didn't recognise the RAM at all and tried to build 16 KiB out of 131,072 flip-flops, and I thought the tool had hung. The fix was four byte-wide RAMs side by side, plus a MAX 10 configuration mode that allows RAM to start with contents at all.
- **One instruction per clock is slow.** Every instruction does everything in a single cycle: fetch, register read, ALU, memory decode, next address. The timing analyser said 40 MHz at best, and the board's clock is 50, so a PLL halves it to 25 MHz. The real fix is a pipeline, which is coming.

Next up: a serial *receiver* and a small bootloader, so I can send a new program over the cable in a second instead of rebuilding the FPGA image for eight minutes. After that, performance counters, a pipeline, caches, and eventually a small matrix-multiply accelerator.

Seeing a CPU I can read every line of print its first primes was a properly good evening.
