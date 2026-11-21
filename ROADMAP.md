# Protocol Emulator ASIC: Roadmap (learning path, process, verification, review prep)

This file is the plan, not a design. It deliberately contains no architecture answers.

## 0. Facts to re-check against the competition page (not yet verified)

I couldn't fetch the competition page from this environment, so every number below came from earlier notes. Confirm each one on the page or the Tiny Tapeout docs, then tick it.

- [ ] Deadline: 18 Jan 2027 (confirm the time zone and what has to be in by then: GDS, write-up, or both)
- [ ] Shuttle and process: IHP 130 nm via Tiny Tapeout, `ttihp-verilog-template`
- [ ] Allowed tile size (8x4?) and the exact `info.yaml` value
- [ ] Cells per tile (~1K?) and total area
- [ ] Pins: 8 in, 8 out, 8 bidir, plus clk and rst_n. Do any pins or a host interface come pre-assigned?
- [ ] Clock: is "60 MHz is OK" a statement on the page or from a FAQ/Discord?
- [ ] I/O round trip ~10 ns, max output ~66 MHz: whose estimate is this?
- [ ] SRAM macros: allowed on this shuttle? What sizes, and how much area does one cost?
- [ ] Judging criteria and weights, required deliverables, rules on AI use, team rules

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
7. **Blind flexibility test**: someone else, docs only, a protocol you never considered. Time how long it takes and log every question they had to ask.
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
| Several protocols at once? | Shown, or explicitly out of scope with a reason |
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
- **Wk 8**: write-up, docs, examples, submit. Leave a buffer of a few days before 18 Jan.
