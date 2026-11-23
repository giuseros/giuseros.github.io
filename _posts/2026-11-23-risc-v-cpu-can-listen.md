---
layout: post
title: "My RISC-V CPU can listen now"
date: 2026-11-23
tags: [riscv, verilog, fpga, ai]
---

[Last time]({% post_url 2026-11-16-risc-v-cpu-on-an-fpga %}), my RISC-V core, [**rv-ai-lab**](https://github.com/giuseros/rv-ai-lab), was running C on an FPGA and printing primes to my laptop. But it could only talk. Whatever I typed went nowhere.

Now it listens. This is a C program running on the board, answering what I type:

```
rv-ai-lab: hello from the DE10-Lite. Type a line; "quit" ends.
> hello, board
  reversed: draob ,olleh   (12 characters)
> never odd or even
  reversed: neve ro ddo reven   (17 characters)
> quit
bye: 2 lines
```

The moment I knew it really worked was when I made a typo, pressed Backspace, and the character disappeared from the screen. That's a round trip from my keyboard, down a wire into the FPGA, through hardware I can read every line of, into a C program, and all the way back.

## The system as it is now

<figure style="margin: 1.5em 0;">
<svg viewBox="0 0 720 350" width="100%" role="img" aria-label="Diagram of the SoC on the DE10-Lite. The RISC-V CPU fetches instructions directly from a 16 KiB block RAM. Its loads and stores go through a bus with a valid/ready handshake, either to the RAM or to the console. The console prints through a UART transmitter, receives through a UART receiver and a 16-byte FIFO, and has an exit register that drives the 7-segment displays. Two wires connect the UART to a PL2303 USB-serial adapter and the laptop." xmlns="http://www.w3.org/2000/svg" style="font-family: inherit; font-size: 13px;">
<style>
.sa-box { fill: var(--bg-card, #fff); stroke: var(--text-muted, #6b7280); stroke-width: 1.2; }
.sa-cpu { fill: var(--accent-soft, #eef0fe); stroke: var(--accent, #4f46e5); stroke-width: 1.6; }
.sa-new { fill: var(--accent-soft, #eef0fe); stroke: var(--accent, #4f46e5); stroke-width: 1.4; }
.sa-grp { fill: var(--bg-subtle, #f7f8fa); stroke: var(--text-muted, #6b7280); stroke-width: 1.2; }
.sa-board { fill: none; stroke: var(--accent, #4f46e5); stroke-width: 1.2; stroke-dasharray: 6 4; }
.sa-t { fill: var(--text, #1a1d23); }
.sa-b { fill: var(--text, #1a1d23); font-weight: 600; }
.sa-m { fill: var(--text-muted, #6b7280); font-size: 11.5px; }
.sa-a { fill: var(--accent, #4f46e5); font-size: 11.5px; font-weight: 600; }
.sa-mono { font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; font-size: 11px; }
.sa-line { stroke: var(--text-muted, #6b7280); stroke-width: 1.4; fill: none; }
.sa-head { fill: var(--text-muted, #6b7280); }
</style>
<defs>
<marker id="sa-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="sa-head"/></marker>
</defs>
<!-- the board -->
<rect class="sa-board" x="12" y="12" width="526" height="326" rx="10"/>
<text class="sa-m" x="24" y="32">DE10-Lite · MAX 10 FPGA · 25 MHz</text>
<!-- CPU -->
<rect class="sa-cpu" x="30" y="55" width="135" height="70" rx="6"/>
<text class="sa-b" x="97" y="82" text-anchor="middle">RISC-V CPU</text>
<text class="sa-m" x="97" y="99" text-anchor="middle">RV32I</text>
<text class="sa-m" x="97" y="114" text-anchor="middle">one instruction a cycle</text>
<!-- RAM -->
<rect class="sa-box" x="30" y="235" width="135" height="70" rx="6"/>
<text class="sa-b" x="97" y="262" text-anchor="middle">RAM · 16 KiB</text>
<text class="sa-m" x="97" y="279" text-anchor="middle">block RAM</text>
<text class="sa-m" x="97" y="294" text-anchor="middle">program and data</text>
<!-- fetch: straight from RAM -->
<line class="sa-line" x1="65" y1="233" x2="65" y2="127" marker-end="url(#sa-arrow)"/>
<text class="sa-m" x="73" y="185">instructions</text>
<!-- bus -->
<rect class="sa-grp" x="212" y="55" width="36" height="250" rx="6"/>
<text class="sa-b" x="230" y="184" text-anchor="middle" transform="rotate(-90 230 180)">bus</text>
<line class="sa-line" x1="167" y1="90" x2="210" y2="90" marker-start="url(#sa-arrow)" marker-end="url(#sa-arrow)"/>
<text class="sa-m" x="188" y="82" text-anchor="middle">data</text>
<line class="sa-line" x1="210" y1="270" x2="167" y2="270" marker-start="url(#sa-arrow)" marker-end="url(#sa-arrow)"/>
<line class="sa-line" x1="250" y1="150" x2="273" y2="150" marker-start="url(#sa-arrow)" marker-end="url(#sa-arrow)"/>
<text class="sa-m" x="230" y="322" text-anchor="middle">valid / ready</text>
<!-- console -->
<rect class="sa-grp" x="275" y="50" width="245" height="190" rx="8"/>
<text class="sa-b" x="290" y="72">console</text>
<text class="sa-m" x="353" y="72">memory-mapped I/O</text>
<text class="sa-t sa-mono" x="290" y="104">print</text>
<rect class="sa-box" x="425" y="86" width="82" height="30" rx="5"/>
<text class="sa-t" x="466" y="105" text-anchor="middle">UART TX</text>
<line class="sa-line" x1="330" y1="101" x2="423" y2="101" marker-end="url(#sa-arrow)"/>
<text class="sa-t sa-mono" x="290" y="145">RX_DATA</text>
<text class="sa-t sa-mono" x="290" y="160">RX_STAT</text>
<rect class="sa-new" x="350" y="133" width="62" height="34" rx="5"/>
<text class="sa-t" x="381" y="148" text-anchor="middle">FIFO</text>
<text class="sa-m" x="381" y="161" text-anchor="middle">16 bytes</text>
<rect class="sa-new" x="425" y="135" width="82" height="30" rx="5"/>
<text class="sa-t" x="466" y="154" text-anchor="middle">UART RX</text>
<line class="sa-line" x1="423" y1="150" x2="414" y2="150" marker-end="url(#sa-arrow)"/>
<line class="sa-line" x1="348" y1="150" x2="340" y2="150" marker-end="url(#sa-arrow)"/>
<text class="sa-a" x="466" y="183" text-anchor="middle">new</text>
<text class="sa-t sa-mono" x="290" y="212">exit</text>
<path class="sa-line" d="M322,208 H400 V253" marker-end="url(#sa-arrow)"/>
<!-- displays -->
<rect class="sa-box" x="300" y="255" width="200" height="50" rx="6"/>
<text class="sa-b" x="400" y="277" text-anchor="middle">7-segment displays</text>
<text class="sa-m" x="400" y="293" text-anchor="middle">the program's exit code</text>
<!-- off the board -->
<line class="sa-line" x1="509" y1="101" x2="561" y2="101" marker-end="url(#sa-arrow)"/>
<line class="sa-line" x1="561" y1="150" x2="509" y2="150" marker-end="url(#sa-arrow)"/>
<text class="sa-m" x="549" y="94" text-anchor="middle">TX</text>
<text class="sa-m" x="549" y="166" text-anchor="middle">RX</text>
<rect class="sa-box" x="563" y="80" width="66" height="90" rx="6"/>
<text class="sa-b" x="596" y="120" text-anchor="middle">USB-</text>
<text class="sa-b" x="596" y="135" text-anchor="middle">serial</text>
<text class="sa-m" x="596" y="152" text-anchor="middle">PL2303</text>
<line class="sa-line" x1="631" y1="125" x2="654" y2="125" marker-start="url(#sa-arrow)" marker-end="url(#sa-arrow)"/>
<rect class="sa-box" x="656" y="80" width="60" height="90" rx="6"/>
<text class="sa-b" x="686" y="120" text-anchor="middle">laptop</text>
<text class="sa-m" x="686" y="137" text-anchor="middle">picocom</text>
</svg>
<figcaption style="text-align: center; font-size: .9em; color: var(--text-muted);">The whole system, all Verilog on the FPGA except the adapter and the laptop. New this time: the UART receiver and its FIFO.</figcaption>
</figure>

The CPU fetches instructions straight from RAM. Its loads and stores go over a small bus with a valid/ready handshake, either to RAM or to the **console**. The console is the only I/O device so far, a handful of addresses that do things instead of storing things. Store a byte to one address and it goes out of the UART as a character. Store a word to another and the program ends, with its exit code on the 7-segment displays. The new part is on the receiving side: a UART receiver, a 16-byte FIFO, and two addresses to read it through.

## Listening is harder than talking

Sending is easy: you decide when each bit starts. Receiving is guesswork. No clock comes with the data, only a line that drops low when a character begins. From that edge the receiver counts its own clock cycles, waits half a bit to land in the middle of the start bit, and then samples once in the middle of every bit after it. The two ends' clocks never agree exactly, so sampling in the middle leaves the most room for them to drift apart.

It also has to avoid inventing characters. A short glitch on the line isn't a start bit, and neither is the middle of a character that was already arriving when the board came out of reset. Every one of those cases has a test now, and a test that breaks the receiver on purpose to make sure the test notices.

## The sender never waits

This was the real lesson. My laptop sends whenever it likes, and nothing on the line can tell it to wait. So anything that arrives while the program is busy has to be kept somewhere until it gets read. That's what the FIFO is for, and 16 bytes is what the classic PC serial chip, the 16550, had too.

A buffer only absorbs **bursts**, though. If the receiver is slower than the sender *on average*, even by a hair, it falls behind a little more with every character, and no buffer is big enough for an endless stream.

I learned that from my own transmitter. To test the receiver, I first built an echo: every character in goes straight back out. It turned out my transmitter rested for one clock cycle between characters: 4,341 cycles per character instead of 4,340, about 0.02% slower than my laptop. A long enough paste would have lost a character. The fix was one flip-flop.

The simulator had hidden it at first. In simulation both ends of the wire had *exactly* the same speed, which never happens in real life. A version of the echo with no buffer at all passed every test. Now the simulated laptop runs 0.2% fast, as a real one might, and the broken version fails.

## Reading can change things

Normally reading memory changes nothing. Reading the receive register does: it hands you the oldest character and removes it from the FIFO. That gives the CPU a responsibility. It must only touch that address when it really means to load from it.

The CPU computes addresses with its ALU, and the result is on the address wires every cycle, even for an ordinary `add`. That's why the bus has a *valid* signal: "this cycle is a real load or store". To prove it matters, I removed the check from the receiver on purpose. The test program, which reads lines the testbench types into the serial line, hung forever. An instruction that merely *computed* the receiver's address had swallowed a character, and the program sat waiting for a character that would never come.

## Quartus surprised me again

I'd assumed the FIFO would be built out of flip-flops, because the logic reads it without waiting for a clock edge, and FPGA block RAM can't do that. Quartus put it into a block RAM anyway. The address it reads from comes straight out of a register, so the tool moved that register into the RAM itself, and added a little logic for the case where the same slot is written and read in the same cycle. Simulation can't check that kind of transformation, which is one more reason to run the real thing.

## What's next

Every new program still means rebuilding the whole FPGA image, which takes eight minutes. Now that the board can listen, the plan is a small **bootloader**: a program that stays in the RAM, receives a new program over the serial line in about a second, and jumps to it.

The code, tests and all, is on GitHub: [**giuseros/rv-ai-lab**](https://github.com/giuseros/rv-ai-lab).
