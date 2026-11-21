# Phase 1 learning guide: Week 1 (23–29 Nov)

This guide covers the first milestone in [ROADMAP.md](../../ROADMAP.md) §5, Week 1:

> protocols, PIO/PRU, PIO exercises + pain list, counter hardened, facts in §0 verified, write-up skeleton

That maps to roadmap sections **1.1** (protocols at the wire level), **1.2** (prior art), **1.4** (Tiny Tapeout flow) and the slice of **1.3** you need to harden a counter.

**What this guide is not.** It doesn't contain an emulator architecture, an ISA, a list of "primitives you'll need", or answers to its own questions. Every question is something you answer in your notebook. If an answer feels obvious, write it down anyway. Writing it is the check.

---

## How to use this guide

1. **Work topic by topic, in order.** Each topic has: *Concepts* (what to learn), *Resources* (free, primary sources first), *Exercises* (small, hands-on), and *You understand it when…* (a checklist you tick only when you can do the thing **without notes**).
2. **Keep a lab notebook** at `notes/week1.md` (or on paper). Date every entry. Record wrong guesses too. They are useful for the write-up's "how I learned" and "AI use" sections.
3. **The 45-minute rule.** If you're stuck on one thing for 45 minutes, write down exactly what you tried, move on, and come back the next day. Week 1 must not spill into week 2 (roadmap §0, "time-box the learning").
4. **Verify with a tool, not with a feeling.** Where possible, a check ends in something that can disagree with you: a simulator, a decoder, a synthesis report.
5. **Using AI.** Fine (and allowed by the rules), but ask it to *quiz* you or *check* your answer, not to give you the answer first. Log each use in the notebook.

### Links

Links were chosen from official docs and well-known references. If one has moved, search for its title, and fix the link here.

---

## Topic 0: Your toolbox (half a day, do first)

You'll use these all week. Install them before anything else so a broken install doesn't block a protocol day.

### Concepts
- Writing a small synchronous Verilog module and a testbench.
- Running a simulation and reading a waveform (VCD) in GTKWave.
- Opening a logic-analyser capture in PulseView and attaching a protocol decoder.

### Resources
- **Icarus Verilog** (simulator): <https://steveicarus.github.io/iverilog/>
- **GTKWave** (waveform viewer): <https://gtkwave.sourceforge.net/>
- **HDLBits** (free browser Verilog exercises with instant checking): <https://hdlbits.01xz.net/>
- **sigrok / PulseView** (logic analyser software and protocol decoders): <https://sigrok.org/wiki/PulseView>, decoder list: <https://sigrok.org/wiki/Protocol_decoders>
- **sigrok-dumps** (real example captures of UART, SPI, I2C and more): <https://github.com/sigrokproject/sigrok-dumps>
- **cocotb quickstart** (you'll use it properly in week 4, but skim now): <https://docs.cocotb.org/en/stable/quickstart.html>

### Exercises
- 0.1 Do the HDLBits sections "Verilog Language → Basics", "Procedures" and "Circuits → Sequential Logic → Latches and Flip-Flops" and "Counters". Stop when counters are green.
- 0.2 Write a 4-bit counter with a synchronous active-low reset and a testbench that dumps a VCD. Open it in GTKWave.
- 0.3 Open any UART capture from sigrok-dumps in PulseView and add the UART decoder. Don't try to understand it yet.

### You understand it when…
- [ ] You can write a clocked `always` block with reset from a blank file, without looking anything up.
- [ ] You can explain, in one sentence each, the difference between `=` and `<=`, and why mixing them in clocked logic causes simulation/synthesis mismatches.
- [ ] You can find the clock edge where your counter wraps in GTKWave and read the value of every bit at that edge.
- [ ] PulseView opens a capture and shows decoded bytes.

---

## Topic 1: UART (1 day)

### Concepts
- Idle level, start bit, data bits (LSB first), optional parity, stop bit(s). What "8N1" means.
- Baud rate vs bit period. Why both ends must agree beforehand (no shared clock).
- How a receiver finds the start of a frame and why it samples **in the middle** of each bit.
- **Oversampling** (e.g. 16×, 8×, 4×, even 1×-with-delay): what it buys and what it costs.
- **Baud-rate error tolerance**: how timing error accumulates across a frame, and how to compute the maximum combined mismatch between two ends.
- Framing errors, break conditions, and what happens if the line glitches during idle.

### Resources
- Analog Devices, *UART: A Hardware Communication Protocol*: <https://www.analog.com/en/resources/analog-dialogue/articles/uart-a-hardware-communication-protocol.html>
- SparkFun, *Serial Communication*: <https://learn.sparkfun.com/tutorials/serial-communication>
- Analog Devices / Maxim, *Determining Clock Accuracy Requirements for UART Communications* (the baud-tolerance maths): <https://www.analog.com/en/resources/technical-articles/determining-clock-accuracy-requirements-for-uart-communications.html>
- sigrok UART decoder page (its options tell you what knobs a real UART has): <https://sigrok.org/wiki/Protocol_decoder:Uart>

### Exercises
- 1.1 **By hand.** Draw the waveform for the byte `0x4B` sent as 8N1, then as 7E1. Label every bit.
- 1.2 **Numbers.** At a 60 MHz clock, how many clock cycles per bit at 9600, 115200 and 1 000 000 baud? Which of those divide evenly? What is the percentage error if you round to the nearest whole number of cycles? (Do it in a spreadsheet. You'll reuse this sheet in week 2's budgets.)
- 1.3 **Tolerance derivation.** Without looking at the Maxim note, derive the maximum baud mismatch for an 8N1 frame when the receiver samples at exactly mid-bit. Then repeat for a receiver that only knows the start edge to within ±1/16 of a bit. *Then* read the note and compare.
- 1.4 **Decode a capture.** Pick a UART capture from sigrok-dumps. Measure one bit period with PulseView's cursors, work out the baud rate, and decode the first three bytes by hand in your notebook. Then turn on the decoder and diff.
- 1.5 **Build a plain UART transmitter in Verilog** (fixed baud, 8N1) with a testbench that checks its output against a simple model in the testbench. This is a *learning* exercise about one fixed-function UART, not a piece of the emulator. Keep it in `scratch/`, not in the design tree.

### You understand it when…
- [ ] You can draw an 8N1 frame from memory, including idle level, and say which edge a receiver syncs on.
- [ ] You can state your derived baud tolerance number and explain *where* the worst-case sample lands.
- [ ] You can explain what oversampling rate you'd want for a given clock and baud, and what you lose by going lower.
- [ ] Your hand decode of 1.4 matches the sigrok decoder byte for byte.
- [ ] Your TX testbench fails if you deliberately send MSB-first. (Try it. A test that can't fail isn't a test.)

---

## Topic 2: SPI (1 day)

### Concepts
- Signals: SCLK, MOSI/COPI, MISO/CIPO, CS/SS. Full duplex: every clock shifts one bit in each direction.
- **CPOL and CPHA → modes 0–3.** For each mode: idle clock level, which edge *launches* data, which edge *samples* it.
- MSB-first vs LSB-first, word sizes other than 8, CS behaviour between words.
- **Controller vs target.** In target mode you sample a clock you don't own. What that means for a design that runs on its own clock: the incoming SCLK is just another asynchronous input.
- Why max SPI target speed is limited by how fast you can *detect* an edge on SCLK, not by your own clock alone. (You'll quantify this in Topic 6 and week 2.)
- Variants to recognise by name only: 3-wire SPI, dual/quad SPI.

### Resources
- Analog Devices, *Introduction to SPI Interface* (clear mode diagrams): <https://www.analog.com/en/resources/analog-dialogue/articles/introduction-to-spi-interface.html>
- SparkFun, *Serial Peripheral Interface (SPI)*: <https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi>
- Wikipedia's SPI article has a good CPOL/CPHA timing figure: <https://en.wikipedia.org/wiki/Serial_Peripheral_Interface>
- A real device datasheet: pick any SPI flash or sensor you own (e.g. a W25Qxx flash) and read its "SPI timing" section.

### Exercises
- 2.1 **By hand.** Draw all four modes for sending `0xA5` controller→target while the target returns `0x3C`. Mark launch and sample edges with different symbols.
- 2.2 **Mode detective.** Find two SPI captures in sigrok-dumps. Work out the mode from the waveform alone, then confirm by setting the decoder's CPOL/CPHA. Set it *wrong* on purpose and note how the decoded output breaks.
- 2.3 **Datasheet reading.** From your chosen device's datasheet, extract: supported modes, max SCLK, setup and hold times. Write them in a small table.
- 2.4 **Think, don't build.** Write half a page: "If my chip ran at 60 MHz and sampled SCLK as an ordinary input, roughly how fast could the external SCLK be before I miss edges?" You'll redo this properly once you've done Topic 6. Keep your first guess so you can see how it changes.

### You understand it when…
- [ ] Given (CPOL, CPHA) you can name the mode, and vice versa, instantly.
- [ ] You can draw all four modes from memory with launch/sample edges correct.
- [ ] Your 2.2 mode identification matches the decoder.
- [ ] You can explain, without notes, why target mode is harder than controller mode.

---

## Topic 3: I2C (1.5 days, the trickiest)

### Concepts
- **Open-drain** lines with pull-up resistors. Why nobody ever drives a 1. What "release the line" means electrically.
- START, STOP and **repeated START**. Why these are the only times SDA may change while SCL is high.
- Address byte (7-bit + R/W), 10-bit addressing (recognise it), ACK/NACK, who drives SDA on the ACK bit for writes vs reads.
- **Clock stretching**: a target holds SCL low. What the controller must do to notice.
- **Arbitration** between two controllers, and why open-drain makes it possible.
- Standard (100 kHz) and Fast (400 kHz) mode timing parameters: t_HD;STA, t_SU;STO, t_BUF, t_LOW, t_HIGH, rise time limits.
- **Spike suppression** (the 50 ns filter in Fast mode) and why it exists.
- Pull-up sizing vs bus capacitance vs rise time.

### Resources
- **NXP UM10204, *I2C-bus specification and user manual*** (the primary source; read sections 3.1 and the timing tables): <https://www.nxp.com/docs/en/user-guide/UM10204.pdf>
- TI SLVA704, *Understanding the I2C Bus*: <https://www.ti.com/lit/an/slva704/slva704.pdf>
- TI SLVA689, *I2C Bus Pullup Resistor Calculation*: <https://www.ti.com/lit/an/slva689/slva689.pdf>
- SparkFun, *I2C*: <https://learn.sparkfun.com/tutorials/i2c>
- sigrok I2C decoder page: <https://sigrok.org/wiki/Protocol_decoder:I2c>

### Exercises
- 3.1 **By hand.** Draw a full transaction: controller writes register address `0x10` to a target at address `0x48`, then a repeated START and a 2-byte read, ending with NACK + STOP. Mark who drives SDA on every bit.
- 3.2 **Spec hunt.** From UM10204, copy the Standard-mode and Fast-mode timing table into your notebook. For each parameter write one sentence on *why* it exists.
- 3.3 **Pull-up maths.** For a 3.3 V bus with 200 pF capacitance at 400 kHz, compute the allowed pull-up range using SLVA689. Then for 100 pF.
- 3.4 **Capture decode.** Decode one I2C capture from sigrok-dumps by hand up to the first data byte (address, R/W, ACK). Diff against the decoder.
- 3.5 **Stretching and arbitration on paper.** Draw (a) a target stretching the clock after the ACK bit, and (b) two controllers starting at the same time with different addresses, showing the bit where one loses.
- 3.6 **Open-drain in Verilog.** In a testbench, model two devices on one wire using `tri1` / pull-up and output-enable (drive 0 or release). Show in GTKWave that either can pull the line low and nobody can force it high. This is about understanding the bus, not about the emulator.

### You understand it when…
- [ ] You can draw START, STOP and repeated START from memory and state the rule that defines them.
- [ ] You can say who drives SDA during every bit of 3.1, including both ACKs and the final NACK.
- [ ] You can explain how a controller detects clock stretching and how it detects losing arbitration, in two sentences each.
- [ ] You can explain why I2C is "open-drain" in terms of output-enable, without the words "open-drain".
- [ ] Your 3.4 hand decode matches the decoder.

---

## Topic 4: Wire-level review and capture practice (half a day)

This closes roadmap §1.1's checks.

### Exercises
- 4.1 **From memory, on blank paper**, draw the timing diagram for: UART 8N1, all four SPI modes, I2C write and I2C read with repeated START. Then compare with your sources and mark every mistake in red.
- 4.2 **Blind decode.** Pick one capture you haven't looked at yet (any of the three protocols). Without the decoder, identify the protocol, its parameters (baud / mode / address) and the first few bytes. Then diff.
- 4.3 **Skim the blind-test candidates** in the roadmap (WS2812, 1-Wire, PS/2, DHT11, DShot, IR NEC, CAN, SWD, JTAG). For each, spend 10 minutes and write *one line* describing how bits are encoded on the wire (e.g. "pulse width", "edge position", "level per clock"). Don't go deeper this week.

### You understand it when (roadmap §1.1 boxes)
- [ ] **Draw every timing diagram from memory**: 4.1 has zero red marks on a second attempt.
- [ ] **Decode a sigrok capture by hand and diff it against the sigrok decoder**: 4.2 matches.
- [ ] You have a one-line encoding description for each blind-test candidate.

---

## Topic 5: Prior art (1.5 days, the most important topic)

Roadmap §1.2 calls this the baseline to beat. The goal is to *feel* what is easy and what is painful in existing programmable-I/O designs. Your opinions here will later justify (or kill) your own design choices, so take notes on friction as it happens, not afterwards.

### Concepts
- **RP2040/RP2350 PIO**: state machines, the instruction set (it's small, learn all of it), side-set, delay cycles, autopush/autopull, shift registers (ISR/OSR), scratch registers, FIFOs and joining, `wait`, `jmp` conditions, clock dividers, IRQ flags between state machines.
- **TI PRU**: a tiny deterministic RISC core with direct GPIO access. How it gets cycle-exact timing without special I/O instructions.
- **XMOS ports**: clocked ports, timed I/O (output at time T), buffered/serialising ports, events.
- **Pure bit-banging** on a normal MCU: what makes it jittery (interrupts, caches, bus contention) and why that's the floor to beat.

### Resources
- **RP2040 datasheet, chapter 3 (PIO)**: <https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf>
- **RP2350 datasheet, PIO chapter** (note what changed vs RP2040): <https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf>
- Raspberry Pi Pico C SDK book, PIO chapter (worked examples, pioasm): <https://datasheets.raspberrypi.com/pico/raspberry-pi-pico-c-sdk.pdf>
- Raspberry Pi Pico Python SDK book (PIO from MicroPython, quickest way to try on hardware): <https://datasheets.raspberrypi.com/pico/raspberry-pi-pico-python-sdk.pdf>
- Official PIO examples (read *after* writing your own): <https://github.com/raspberrypi/pico-examples/tree/master/pio>
- **No hardware?** PIO emulators: `rp2040pio` (Java, with docs) <https://rp2040pio-docs.readthedocs.io/en/latest/introduction.html>, the Python `rp2040-pio-emulator` <https://pypi.org/project/rp2040-pio-emulator/>, or Wokwi's Pico simulator <https://docs.wokwi.com/parts/wokwi-pi-pico>
- **PRU Cookbook** (BeagleBoard): <https://docs.beagleboard.io/books/pru-cookbook/index.html>
- **XMOS**: *Introduction to XS1 ports* <https://www.xmos.com/download/Introduction-to-XS1-ports(3).pdf> and AN03001 *XCORE Clocked Input and Output* <https://www.xmos.com/documentation/XM-015254-AN/html/doc/rst/an03001.html>

### Exercises
- 5.1 **Learn the PIO ISA cold.** Make a one-page cheat sheet of every PIO instruction in your own words. Then put it away and write it again from memory.
- 5.2 **UART TX in PIO** (write it yourself first, then compare with pico-examples). Run it on a Pico (look at the output with a logic analyser or a USB-UART adapter) or in an emulator.
- 5.3 **UART RX in PIO.** Same process. Pay attention to how you find mid-bit.
- 5.4 **I2C in PIO.** This one is meant to be hard. Try for one session (max ~3 hours) before reading the official example. Note precisely where you got stuck.
- 5.5 **Read a PRU example** (e.g. a cookbook chapter that toggles or reads a pin at fixed timing) and an XMOS clocked-port example. You don't need to run them. For each, write what the programmer has to do to get exact timing.
- 5.6 **Pain list (one page).** For each of 5.2–5.5, list what was awkward. For every item, write the *feature or instruction that would have removed the pain* and *what that feature would cost* (area, complexity, verification effort). Be specific: "I had to burn N instructions to do X" is better than "X was annoying".
- 5.7 **The RISC-V paragraph.** One paragraph: why not put a tiny RISC-V core plus GPIO on the chip? Argue both sides honestly. A reviewer will ask (roadmap §4).

### You understand it when (roadmap §1.2 boxes)
- [ ] **UART TX/RX and I2C written in PIO assembly and run on hardware or a simulator.**
- [ ] **One-page pain list** exists, every entry has a "would remove it" and a "would cost" column.
- [ ] **One paragraph on the RISC-V question** exists and contains at least one number (cycles, area, or jitter), even if rough.
- [ ] Without notes, you can explain how PIO gets cycle-exact timing, how the PRU does, and how XMOS does. Three different answers.
- [ ] You can explain what side-set and autopush are *for*, in terms of instruction count saved.

---

## Topic 6: Just enough ASIC-style design for week 1 (half a day)

Roadmap §1.3 is mostly a week-2 topic. This week you only need what the counter hardening and the SPI-target thinking require.

### Concepts
- **Reset**: on silicon, flops power up in an unknown state. Why every flop that matters needs `rst_n`, and why `initial` blocks don't exist on a chip.
- **Unintended latches**: how an incomplete `if`/`case` in combinational logic creates one, and how to spot it in a Yosys log.
- **Metastability and the 2-flop synchroniser**: what it protects against, and what it costs in latency.
- **Tiny Tapeout bidirectional pins**: `uio_out`, `uio_oe`, `uio_in`. How output-enable lets you emulate open-drain.

### Resources
- Clifford Cummings, *Clock Domain Crossing (CDC) Design & Verification Techniques Using SystemVerilog* (sections 1–4 are the classic synchroniser explanation): <http://www.sunburst-design.com/papers/CummingsSNUG2008Boston_CDC.pdf>
- Tiny Tapeout HDL intro and pin conventions: <https://tinytapeout.com/hdl/> and <https://tinytapeout.com/specs/>
- Yosys (open-source synthesis, used by the TT flow): <https://github.com/YosysHQ/yosys>

### Exercises
- 6.1 Write a combinational block with a deliberately incomplete `case`, run `yosys -p "read_verilog x.v; synth; stat"` and find the latch in the log. Fix it and confirm the latch is gone.
- 6.2 In simulation, feed an asynchronous square wave through a 2-flop synchroniser. Measure, in clock cycles, the delay from an input edge to the synchronised output edge. What's the minimum and maximum?
- 6.3 Redo your Topic 2 exercise 2.4 using what you measured in 6.2. Write down how your SPI target speed estimate changed and why.

### You understand it when…
- [ ] You can explain why "flops start at 0" is false on real silicon and what goes wrong if you assume it.
- [ ] You can spot a latch in a Yosys report.
- [ ] You can give the min/max edge latency of a 2-flop synchroniser in cycles and explain where the range comes from.
- [ ] You can explain how to make a TT bidir pin behave like an open-drain line using only `uio_out` and `uio_oe`.

---

## Topic 7: Tiny Tapeout / IHP flow, counter hardened (1 day, start early in the week)

Roadmap §1.4 says this must be done **in week 1**. GitHub Actions runs can take a while, so kick this off on day 1 or 2 and let it run while you study protocols.

### Concepts
- The `ttihp-verilog-template` layout: `src/`, `test/`, `info.yaml`, `docs/`.
- `info.yaml`: top module name, tile count (**8x4** per the rules), clock frequency, pinout descriptions.
- What the GitHub Actions do: test, GDS (hardening), precheck, docs, and what "green" means for each.
- Local hardening, for when you hit the Actions time limit.
- Gate-level simulation (GLS): simulating the post-synthesis netlist instead of your RTL. What kinds of bugs it catches that RTL sim doesn't.
- The demo board: what the RP2350 does for your design (clock, reset, driving inputs, reading outputs) and the MicroPython tooling.

### Resources
- Template repo (main branch, per the rules): <https://github.com/TinyTapeout/ttihp-verilog-template>
- Tiny Tapeout HDL templates page: <https://tinytapeout.com/hdl/templates/>
- Local hardening guide: <https://tinytapeout.com/guides/local-hardening>
- Tiny Tapeout guides index (demo board, testing, GLS): <https://tinytapeout.com/guides/>
- Tiny Tapeout GitHub org (look for the demo board MicroPython firmware and the GDS action): <https://github.com/TinyTapeout>
- Ken Pettit's SRAM walkthrough is linked from the competition page. Skim it this week, study it in week 2.

### Exercises
- 7.1 Create a repo from the template (it can stay private). Replace the example with your Topic 0 counter, widened so it uses the outputs. Set the tile size and clock in `info.yaml`.
- 7.2 Push and get **all actions green**. Every time one fails, note the error and the fix in your notebook. These notes will save you hours in week 7.
- 7.3 Download the GDS artifact and open it in the viewer the action links to. Find your design's area utilisation in the reports.
- 7.4 Run the gate-level test (the template's test setup supports it; follow the TT guide). Make it pass.
- 7.5 **Plant a bug** that RTL sim passes but GLS or synthesis would expose (hint: think about Topic 6's reset and latch concepts). Confirm which stage catches it.
- 7.6 **Prediction game.** Before each hardening run, write down your guess for the cell count. After the run, record the actual. Start the area spreadsheet for roadmap §2 now.

### You understand it when (roadmap §1.4 boxes)
- [ ] **Trivial counter hardened at the target tile size, GDS green, in week 1.**
- [ ] **Gate-level sim of that counter passing.**
- [ ] You can explain what each GitHub Action does and what you'd look at first if each one went red.
- [ ] You can explain in one paragraph what the RP2350 on the demo board does for your design at power-up and how a program would get onto the chip from it (from the docs, not a guess).
- [ ] Your 7.5 bug was caught by something other than RTL sim, and you can say why RTL sim missed it.

---

## Topic 8: Close out week 1 (half a day, day 7)

### Exercises
- 8.1 **Verify the facts in roadmap §0.** For each row in the rules table, find the sentence on the competition page or TT docs that backs it up and paste the quote and date into your notebook. Mark anything that changed.
- 8.2 **Write-up skeleton.** Create `docs/writeup.md` with headings only (e.g. Introduction, Prior art and baseline, Requirements, Architecture, Programming model, Toolchain, Verification, Results, Limits, AI use, Acknowledgements). Paste your pain list and RISC-V paragraph into the right sections as drafts.
- 8.3 **Self-review.** Re-run every "You understand it when…" box above *without notes*. Anything you can't tick goes into a "carry-over" list with a time box for week 2.
- 8.4 **Update the roadmap tracker**: tick the week-1 boxes in ROADMAP.md and the roadmap doc.

### You understand it when…
- [ ] Every row of §0 has a source quote or is marked "unverified" with a reason.
- [ ] The write-up skeleton exists in the repo.
- [ ] The carry-over list has fewer than five items, each with a time box.

---

## Suggested day plan (23–29 Nov)

This is a suggestion. Move things around, but keep Topic 7 early because CI is slow.

| Day | Main | In the background |
|---|---|---|
| Mon 23 | Topic 0 toolbox, start Topic 7 (template repo + counter) | First GDS run |
| Tue 24 | Topic 1 UART | Fix CI failures from Mon |
| Wed 25 | Topic 2 SPI, Topic 6 | Topic 7 GLS |
| Thu 26 | Topic 3 I2C | Topic 7 bug planting |
| Fri 27 | Topic 3 finish, Topic 4 review | |
| Sat 28 | Topic 5 prior art (PIO exercises) | |
| Sun 29 | Topic 5 pain list + RISC-V paragraph, Topic 8 close-out | |

---

## What's deliberately left for later

- Requirements matrix, budgets, choosing a paradigm, ISA v1: **week 2** (roadmap §2 steps 0–3). Resist starting them this week. The pain list and the numbers you collected are their inputs.
- STA, critical paths, SRAM macros vs flops: week 2 (roadmap §1.3).
- cocotb in depth, SymbiYosys, CI design: week 4 onwards (roadmap §1.5).
