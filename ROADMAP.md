# Protocol Emulator ASIC: Roadmap (learning path, process, verification, review prep)

This file is the plan, not a design. It deliberately contains no architecture answers.

## 0. Rules, checked against the competition page (post dated 10 Sep 2026, updated 9 Nov 2026; re-checked 21 Nov 2026)

| Item | What the page says | Still open |
|---|---|---|
| Deadline | Submit by **18 Jan 2027**. The page itself gives no time zone or hour; we confirmed **IST** | Internal target: submit by **16 Jan** as a buffer. Submission form is still to be posted closer to the deadline |
| Process | IHP 130 nm **CMOS5L** via Tiny Tapeout. `ttihp-verilog-template` main branch (no `cmos5l` branch needed) | — |
| Area | **8x4 tiles** (set in `info.yaml`), confirmed final. ~202 × 155 µm per tile, ~1.0 mm² total | "~1K logic cells per tile" is their *rough* estimate. Measure it yourself |
| Pins | 24 digital pins: 8 in, 8 out, 8 bidir, plus clk and rst_n. No analog pads. 3.3 V I/O on the demo board | No pre-assigned host interface |
| Program loading | **Part of your design.** The demo board's RP2350 is the natural host, but any documented host works | How the program gets in, and how many pins that costs |
| I/O speed | ~10 ns round trip, ~66 MHz max output | Tiny Tapeout *estimates*. No CMOS5L silicon has been measured yet. Leave margin |
| Clock | No target. The RP2350 on the demo board supplies it. **60 MHz is confirmed OK** | Pick a frequency that closes timing after place-and-route |
| SRAM | IHP SRAM macros allowed, more than one, as long as everything fits and passes precheck. The page notes SRAM can be more area-efficient than flops for instruction memory. See Ken Pettit's walkthrough | Macro area vs flops: measure it |
| Analog | Allowed inside the design (e.g. a PLL) if it passes precheck | — |
| External parts | Allowed on a Pmod (pull-ups, transceivers, level shifters). **You must document them** | I2C pull-ups, USB/Ethernet transceivers |
| Architecture | Doesn't have to be a CPU (CPU-style is just the most common approach). An eFPGA, shared-pipeline engines or anything else is fine, as long as it can be reprogrammed after fab to support new protocols. The software side is required too: assembler/toolchain (or bitstream flow) plus example programs. Suggested inspiration: RP2040 PIO, TI PRU | — |
| Protocols | UART, SPI, I2C. Stretch: LS USB, 10 Mbit Ethernet. Also suggested: JTAG, SWD, PS/2, CAN | No fixed speeds. Targets: common baud rates, SPI modes 0–3, I2C 100/400 kHz |
| Pluses | Running protocols concurrently. Controller *and* target roles (both "a plus but not required"). For USB/Ethernet, the more line coding, CRC and clock recovery on-chip, the better. Unique functionality and novel design/verification methods are of interest | — |
| Reset | Flops don't start at 0, so use `rst_n`. Deselect powers the design down, so reload the program after reselect | — |
| Flow | Tiny Tapeout flow and precheck only; no extra sign-off. Use local hardening + custom GDS action if you hit the 6 h Actions limit. GLS strongly recommended. An FPGA is suggested for testing RTL first, if you have one | — |
| Deliverables | A **link to a public repo** with: RTL, tests, passing GDS/precheck, architecture + instruction-set docs, assembler/toolchain + example programs, verification write-up. A video/demo is optional (reviewed if provided) | Repo can stay private until you submit |
| Judging | No fixed weighting. **Flexibility first** (can it be reprogrammed for protocols it wasn't designed for?), then protocols demonstrated, unique capabilities, design + verification. **Write-up weighed heavily** (readability, clarity on new techniques). Fewer protocols + strong architecture/verification can win | Judges aren't named. Jane Street can't review designs before the deadline, but questions can go to asic-competition@janestreet.com |
| Verification | They're interested in formal, constrained random, AI-assisted verification. Tests that run on open-source tools (cocotb) help them reproduce your results | — |
| AI | Allowed. Describe how you used it in the write-up | — |
| HDL | Any. Jane Street uses Hardcaml, but it's not required | — |
| Teams | Students and professionals, any team size, one entry per team | **Entering solo** |
| Outcome | Winners announced early Feb. ~4 designs taped out on the March 2027 CMOS5L shuttle (subject to the foundry schedule). Winners get the chip on a demo board and work with Jane Street on small non-functional fixes (e.g. re-hardening) | — |

- [x] Fill in the sign-up form (no confirmation email is sent)
- [x] Deadline time zone confirmed: IST
- [x] Team decision: solo

### Solo-entry implications
- **Scope.** The organisers recommend teams because of the workload. Solo, the MVP is UART + SPI + I2C done very well, plus one blind protocol. Stretch protocols only after the week-7 freeze criteria are met in budget.
- **The blind test needs someone who's not you.** That doesn't make them a teammate. Ask a labmate or friend to spend one afternoon writing a protocol from your docs alone, and credit them in the write-up. Fallback: write it yourself from the docs alone, at least two weeks after you last touched the ISA, and log every time you had to read the RTL.
- **No second pair of eyes on reviews.** Make up for it with automation: CI on every push, formal properties, and the lockstep tests. Do a written self-review against §4 at the end of weeks 3, 5 and 7.
- **Time-box the learning.** Weeks 1–2 are learning plus budgets. Don't let prior-art reading spill into week 3.

## 1. Learning path (each stage ends with a check)

### 1.1 Protocols at the wire level
- UART: framing, mid-bit sampling, oversampling, how much baud error you can tolerate.
- SPI: modes 0–3, launch vs sample edge, controller vs target. **Target mode** means sampling a clock you don't own.
- I2C: open-drain, START/STOP, ACK/NACK, clock stretching, arbitration, repeated START, spike filtering.
- Stretch goals: low-speed USB (NRZI, bit stuffing, SE0/EOP, differential), 10BASE-T (Manchester, NLP).
- Blind-test candidates: WS2812, 1-Wire, PS/2, DHT11, DShot, IR NEC, CAN (bit-level), SWD, JTAG.
- [ ] Draw every timing diagram from memory
- [ ] Decode a sigrok capture by hand and diff it against the sigrok decoder

### 1.2 Prior art (most important)
- RP2040/RP2350 PIO, TI PRU, XMOS ports. Also look at small MCUs doing pure bit-banging. That's your **baseline to beat**.
- [ ] UART TX/RX and I2C written in PIO assembly and run on hardware or a simulator
- [ ] One-page pain list: what was awkward, and which instruction or feature would have removed the pain
- [ ] One paragraph: why not just put a tiny RISC-V core plus GPIO on the chip? (A reviewer *will* ask.)

### 1.3 ASIC-style digital design
- Explicit resets, no reliance on power-up values, no unintended latches.
- Synchronisers, metastability, and what a 2-flop synchroniser does to **max sampled input frequency** and edge latency.
- Tristate and output-enable on Tiny Tapeout bidir pins. How open-drain is emulated with OE.
- Setup/hold, STA reports, critical paths, fmax.
- Area estimation: flop vs mux vs memory cost. Why flop-based instruction memory gets expensive quickly. Latch-RAM vs SRAM-macro trade-offs.
- [ ] Predict Yosys cell count within ~30%
- [ ] Name the critical path from a real STA report and explain how you'd shorten it

### 1.4 Tiny Tapeout / IHP flow
- Template, `info.yaml`, precheck, GDS action, local hardening, gate-level sim.
- The demo board: what the RP2350 does (clock, reset, loading data in, reading results back) and its MicroPython/SDK tooling.
- [ ] Trivial counter hardened at the target tile size, GDS green, **in week 1**
- [ ] Gate-level sim of that counter passing

### 1.5 Verification tooling
- cocotb (Icarus/Verilator), Python reference models, constrained random, functional coverage.
- SymbiYosys: assert/assume/cover, BMC vs k-induction, and why an unbounded proof matters.
- CI: tests run on every push (GitHub Actions).
- [ ] Plant 5 different bugs. Record which layer catches each one. Strengthen any layer that misses one.

### 1.6 Toolchain
- Assembler (Python), ISS. Decide early whether the ISS is **cycle-accurate**. Lockstep comparison only works if it is.
- [ ] ISS runs a UART program and dumps a VCD you can open in GTKWave, before any RTL exists

## 2. How an expert runs the project

0. **Choose the paradigm deliberately.** The rules accept a CPU, multiple engines, an eFPGA or something else. Write a one-page comparison of at least three approaches against flexibility, area and verifiability before committing. (Everything below says "ISA". Read it as "programming model" if you don't go CPU-style.)
1. **Requirements matrix**: protocols × primitives, including protocols you won't implement. Find the smallest set of independent primitives that covers it.
2. **Budgets before design**:
   - Cycles per bit (or half-bit) at the chosen clock, for every row.
   - Max *external* clock you can sample, given synchroniser latency (SPI target, I2C at 1 MHz).
   - Host bandwidth: can data move between host and chip at line rate without the FIFO running dry or overflowing?
   - Pin budget: how many pins the host/loading interface uses vs how many are left for protocols.
   - Area: instruction storage, per-engine cost, assists. Fill in a spreadsheet and update it after every synthesis run.
3. **Decision log**: one paragraph per decision, listing the alternatives you rejected and why, with numbers.
4. **Software loop**: ISA doc → ISS → assembler → programs → revise. Freeze the ISA only when every target protocol *and* at least one blind-test protocol fits.
5. **RTL against the golden model**: lockstep, cycle by cycle.
6. **Harden weekly**: track area, slack and utilisation as a time series. They go in the write-up.
7. **Blind flexibility test**: someone else (a friend or labmate; see the solo notes in §0), docs only, a protocol you never considered. Time how long it takes and log every question they had to ask.
8. **Scope control**: define an MVP and a freeze date. Write down what you will cut first if you're late. Only start stretch protocols once the budgets say they fit.

## 3. Verification checklist
- [ ] Unit tests per block
- [ ] Formal: no lost/duplicated FIFO data, PC always in range, pins only driven when enabled, known state after reset, determinism (same program + inputs → same cycle trace)
- [ ] ISS vs RTL lockstep on every example program
- [ ] Protocol checks against independent peers (cocotb peers, or real devices on an FPGA)
- [ ] Constrained random: baud error, SPI modes, stretching, glitches, back-pressure, random reset points
- [ ] Corner cases: reset mid-transfer, reload after deselect, FIFO full/empty boundaries, max/min clock dividers
- [ ] Gate-level sim on the hardened netlist (with SDF if possible)
- [ ] Coverage: every instruction, every branch condition, every pin mode, every FIFO state
- [ ] Measure worst-case output-edge jitter and input-to-action latency, and quote the numbers
- [ ] Optional: FPGA bring-up with a real sensor and a USB-UART bridge

## 4. Reviewer questions (prepare the evidence before you write)

| Question | Evidence to have ready |
|---|---|
| What can't it do, and why? | Limits table with cycle/bandwidth numbers |
| Why this ISA and not PIO / a RISC-V core? | Decision log, baseline comparison |
| Is timing deterministic? Prove it. | Fixed-cycle spec + formal property + trace |
| Max external clock you can follow (SPI target)? | Synchroniser analysis + sim |
| Async inputs and glitches? | Synchroniser/filter design, latency figure |
| Worst-case output jitter? | Gate-level measurement |
| Can the host keep up at line rate? | Bandwidth calc + back-pressure test |
| What happens on FIFO overflow or underflow? | Specified behaviour + test |
| Several protocols at once? Controller and target roles? | Both are listed as pluses. Show them, or explain why they're out of scope |
| For USB/Ethernet, how much runs on-chip (line coding, CRC, clock recovery)? | Breakdown of on-chip vs host work |
| What external parts does it need? | Documented Pmod / pull-up / transceiver list |
| Can we rerun your tests? | Open-source (cocotb) tests and a one-command run |
| Area split, critical path, fmax? | Synthesis/STA reports, area time series |
| How do you know RTL matches the spec? | Lockstep, formal, coverage |
| What bug could your verification miss? | Honest list of what you didn't verify, and the residual risk |
| New protocol in an afternoon? | Blind-test log |
| Reset/deselect? | Reload flow + tests |
| AI use and how you checked it? | AI-use section |

## 5. Eight-week plan
- **Wk 1 (23–29 Nov)**: protocols, PIO/PRU, PIO exercises + pain list, counter hardened, facts in §0 verified, write-up skeleton
- **Wk 2**: requirements matrix, all budgets, ISA v1, decision log started
- **Wk 3**: ISS + assembler, UART/SPI/I2C + one blind protocol on the ISS, ISA frozen
- **Wk 4–5**: RTL, unit tests, lockstep, CI, harden every week
- **Wk 6**: formal, random, GLS, blind test, stretch goal only if budgets allow
- **Wk 7**: close timing and area, freeze, precheck green
- **Wk 8 (11–18 Jan)**: write-up, docs, examples. **Submit by 16 Jan**, leaving 2 days of buffer before the 18 Jan IST deadline.

## Sources and links
- [Competition announcement](https://blog.janestreet.com/protocol-emulator-asic-competition/)
- [Tiny Tapeout IHP Verilog template](https://github.com/TinyTapeout/ttihp-verilog-template)
- [Tiny Tapeout docs](https://tinytapeout.com) · [Demo board specs](https://tinytapeout.com/specs/pcb-etr/) · [Pinouts](https://tinytapeout.com/specs/pinouts/)
- [Local hardening guide](https://tinytapeout.com/guides/local-hardening/) · [FPGA vs ASIC guide](https://tinytapeout.com/hdl/fpga_vs_asic/)
- [SRAM macro walkthrough (Ken Pettit)](https://kdp1965.github.io/ihp-um-janestreet-prism/ihp_sram_macro.html)
- [Hardcaml](https://hardcaml.org) (optional)
- Questions: asic-competition@janestreet.com
