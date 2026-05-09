# Hardware Managing System — Writeup

**Flag:** `HTB{BCKD000RD_HRDWR3}`
**Category:** Hardware / Reverse engineering
**Files:** `CtfTask.v` (Yosys-synthesized Verilog netlist), `VCtfTask` (Verilator simulation binary), `notes.txt`

## TL;DR

The synthesized netlist contains a sticky latch (one bit of an obfuscated 8-bit register) that, when set, forces the master-shutoff signal low — bypassing the temperature comparator entirely. Find an input sequence that arms the latch and the emergency shutdown will silently fail when temperature exceeds 100, causing the harness to print the flag.

## Step 1 — Understand the harness

Run the binary with no input:

```
$ ./VCtfTask
...
Example Input: 001003004099123321999000

Input:
[1] Input: '...', Power Level: '000'
[1] Before Temperature 020 => After Temperature 016
...
[8] Testing Emergency Shutdown:
[8] Input: '255', Power Level: '255'
[8] EMERGENCY SHUTDOWN ACTIVATED!
TEST SUCCESSFUL
```

What it's doing:

- Reads up to **21 characters** from stdin (loops while index ≤ 20, stops on newline).
- Splits them into **7 rounds of 3 characters each**. Each character is sent to the hardware as a 7-bit `io_input` value, with `io_inputDone` strobed high then low (two clock ticks per character).
- After each round of 3 characters, it calls `emulateTemperature(i)` — this reads `io_power` back from the hardware, updates the simulated temperature using `temp = clamp(temp · (power/255 + 0.8), 0, 255)`, and prints the result.
- After all rounds, it sends one final input (`"255"`) as a stress test, runs **100 settling ticks**, and checks the final state. The "Power Level" printed at round 8 is whatever `io_power` happens to be — it is *not* a second user input.

Win condition (from `strings VCtfTask`):

```
TEST FAILED, EMERGENCY SHUTDOWN WAS NOT ACTIVED!
INPUT '%s' NEEDS FURTHER ANALYSIS
```

The harness fails the test (and prints the flag) iff `io_power != 0` **and** the simulated temperature is above 100 at the end. In other words: we need the hardware to keep the system powered while the temperature is dangerous.

A few thousand random inputs all give `TEST SUCCESSFUL`, so the trigger isn't going to fall out of brute force — it's hidden in the hardware.

## Step 2 — Spot the obfuscation

`CtfTask.v` is ~4,700 lines of LUT primitives, MUX cells, and FDCE/FDPE flip-flops. Yosys normally preserves human-readable signal names from the source RTL, but here every meaningful register has been renamed to a 64-character hex string that looks like a SHA-256 digest:

```
\_02b4263a83694edec…[3:0]   4 bits   — reg_A (bits 3:0), lower half of input capture register
\_211d3dff7ef1a9f1f0…[7:0]  8 bits   — *** the backdoor latch lives here ***
\_32351fa08cb046335f…[6:0]  7 bits   — reg_B, input capture register
\_57bbd766e465ab9419…[7:0]  8 bits   — power register
\_7e9c8b61c697a43800…[1:0]  2 bits   — 2-bit state counter (00→01→10→11→00, one cycle per round)
\_913e17b33106a7bcb3…[6:0]  7 bits   — reg_A (bits 6:4), upper half of input capture register
\_9cc539da16a5cc5d21…       1 bit    — registered inputDone
\_c3f22af6d7a88d0aa3…[6:0]  7 bits   — reg_C, input capture register
```

Hash-like names in synthesized RTL are an immediate red flag. They aren't the result of normal synthesis — somebody renamed signals deliberately to make the netlist hard to read.

## Step 3 — Trace `io_power` backwards

Find what drives `io_power[i]` (`CtfTask.v:3471–3509`). Each bit is a 2-input LUT with `INIT=4'b0100`, which is the boolean function `(~I0) & I1`:

```
io_power[i] = (~_270_) & power_register[i]
```

So `_270_` is the master shutoff. When `_270_=1`, every bit of `io_power` is forced to 0 regardless of `power_register`.

Now find what drives `_270_` (`CtfTask.v:3068`):

```
_270_ = (~_271_) & (~_211d3dff…[7])
```

`_271_` is the temperature-status signal. Its LUT inputs are `io_temp[0..7]` — exactly what you'd expect from a "is the temperature safe?" comparator built directly from the temperature bus.

But there is a **second** term: a register bit `_211d3dff…[7]`. Both terms are ANDed together, so:

```
        _211d3dff…[7] == 1   ⇒   _270_ = 0   (shutoff disabled, regardless of temperature)
        _211d3dff…[7] == 0   ⇒   _270_ = ~_271_  (legitimate temperature behavior)
```

That extra register bit is the backdoor. Visually:

```
       io_temp[0..7] ──► [temp-status LUT] ──► _271_ ──┐
                                                       ▼
                                                     ~_271_ ──┐
                                                              & ──► _270_ ──► forces io_power = 0
                                              ~_211d3dff…[7] ──┘
                                                     ▲
                              user input ──► [unlock logic] ──► register bit (latched)
```

## Step 4 — Trace the latch

`_211d3dff…[7]` is an FDCE flip-flop (`CtfTask.v:4631–4636`), with its D input driven by a single LUT6 `_482_` (`CtfTask.v:3047`):

```
LUT6_482(I0=_265_, I1=_269_, I2=_226_, I3=_261_, I4=_213_, I5=_211d3dff…[7])
```

I5 is the register's *own* current value — the LUT6 implements a self-feedback path. Expanding the mux tree of the matching paramod LUT6 module (it's just nested ternaries) and substituting I5=0:

```
when current value = 0:   next value = _226_ & _261_
```

So **the latch arms (and stays armed) when `_226_` and `_261_` are simultaneously 1 on a clock edge.** Both `_226_` and `_261_` are themselves combinational functions of the 2-bit state counter, the input capture registers, and the registered inputDone strobe — i.e., functions of *what the user has typed and when*.

That's the trigger. The challenge reduces to: find an input sequence such that on some cycle, both `_226_` and `_261_` are 1.

## Step 5 — Find the unlocking input

Hand-evaluating the LUTs back to the inputs across the full input phase (~52 ticks: 3 reset + 42 character + 7 temperature-settle) is impractical. The realistic approach is to mechanize it.

### Build a netlist simulator

Parse the gates Yosys emits — `LUT2/3/4/5/6` (each with an `INIT` parameter or, for LUT6, an expanded mux body), `FDCE` (resets to 0), `FDPE` (presets to 1), `MUXCY`, `MUXF7/F8`, `XORCY` — into a simple Python evaluator. For each `LUT*` paramod module, compute the truth table once by symbolically evaluating its body for all 2ⁿ input combinations. Then per clock cycle:

1. Set `clk`, `reset`, `io_inputDone`, `io_input[6:0]`, `io_temp[7:0]` for that cycle.
2. Iterate gate evaluation until wires settle.
3. On the rising edge, latch each FDCE/FDPE D input into Q.

Reset behavior matches what the binary does in `reset()`: assert `reset` for three ticks, then deassert. After that, each character is two ticks (`io_inputDone=1` then `io_inputDone=0`, with `io_input` held the whole time).

### Solve

Two reasonable strategies:

- **SMT.** Unroll the simulator into a Z3 instance — one Bool per wire per cycle, gate constraints from the truth tables, FDCE next-state equations, primary inputs as free variables — and ask Z3 to make `_211d3dff…[7]` true at the final cycle. The model that comes back tells you exactly what bytes to send each round.
- **Guided enumeration.** Run the Python simulator natively (much faster than spawning the binary), and search per-character: try all 128 values of `io_input` for character 1, lock in whichever advances `_226_`/`_261_` most, then do the same for character 2, and so on. The 2-bit state counter (`_7e9c8b61`) steps deterministically with each new character (00→01→10→11→00, one full cycle per 4-byte round), so each character has a fixed "slot" in the state machine — making the search space 128 per character, not 128²¹.

Two pitfalls to watch out for when modeling this:

1. **Don't tie `io_temp` to 0** in your model. The harness drives `io_temp` with the simulated temperature each cycle, and although the documented data path from `io_temp` only goes through `_271_`, pinning it to a constant changes downstream resolution and can rule out otherwise-valid states. Either thread the temperature physics through your simulator or leave `io_temp` free in the SMT formulation.
2. **Cover enough cycles.** Each character is 2 ticks, plus 3 ticks of reset, plus the per-round `tick()` calls inside `emulateTemperature`. Make sure your cycle window covers the entire input phase before you ask Z3 to find the latch high.

### Worked example — decoding Round 1 (bit[4])

To make the decode step concrete, here is the full derivation for the first unlock stage.

The FDCE `_736_` (`CtfTask.v:4610–4616`) stores `_211d3dff[4]`, with D input wired to `_193_` (`CtfTask.v:2909–2914`):

```
LUT3 _464_ (INIT=8'11111000)   I0=_213_  I1=_264_  I2=_266_
```

`INIT=8'11111000` in binary is `11111000`. Bit n of INIT is the output when `{I2,I1,I0}=n`; bits 3–7 are 1, so:

```
_193_  =  _266_  |  (_213_ & _264_)
```

`_266_` is a MUXF8 (`CtfTask.v:2963–2968`) with select = `_211d3dff[4]`. When the bit is currently 0, `_266_` outputs a deep comparison result that evaluates to 0 in normal state; when already 1 it outputs 1, creating the self-latch. The **initial arming** condition therefore reduces to:

```
_193_  =  _213_ & _264_
```

**`_264_` — five simultaneous register comparisons** (`CtfTask.v:2886–2892`):

```
LUT5 _461_ (INIT=32'10000000000000000000000000000000)   I0..I4 = _234_, _235_, _236_, _237_, _238_
```

Only bit[31] is set (= 1 only when all five inputs are 1). Decoding each sub-signal:

| Signal | Gate | INIT | Firing condition | Required bits |
|--------|------|------|-----------------|---------------|
| `_234_` | LUT4 `:2237` | `16'0000000100000000` | bit[8]=1 → `{I3,I2,I1,I0}=1000` | reg_A[1]=1; reg_A[3]=reg_A[2]=reg_A[0]=0 |
| `_235_` | LUT4 `:2244` | `16'0000000100000000` | same | reg_B[2]=1; reg_B[3]=reg_B[1]=reg_B[0]=0 |
| `_236_` | LUT3 `:2251` | `8'00010000` | bit[4]=1 → `{I2,I1,I0}=100` | reg_C[6]=1; reg_C[4]=reg_C[5]=0 |
| `_237_` | LUT6 `:2257` | module body: `O = ~I0 & ~I1 & ~I2 & I3 & I4 & I5` | — | reg_B[4]=1, reg_B[6]=1, reg_A[6]=1; reg_A[4]=reg_A[5]=reg_B[5]=0 |
| `_238_` | LUT5 `:2266` | `32'd16777216` (=2²⁴, bit[24]=1) | `{I4,I3,I2,I1,I0}=11000` | `_211d3dff[0]`=1 (FDPE preset, always 1 after reset); reg_C[3]=1; reg_C[2]=reg_C[1]=reg_C[0]=0 |

Assembling all constraints into 7-bit register values:

```
reg_A [6:0]  =  1 0 0 0 0 1 0  =  0b1000010  =  66  =  'B'
                ↑         ↑
             [_237_]   [_234_]

reg_B [6:0]  =  1 0 1 0 1 0 0  =  0b1010100  =  84  =  'T'
                ↑   ↑   ↑
             [_237_][_237_][_235_]

reg_C [6:0]  =  1 0 0 1 0 0 0  =  0b1001000  =  72  =  'H'
                ↑       ↑
             [_236_] [_238_]
```

**`_213_` — the timing strobe** (`CtfTask.v:1749–1752`):

```
LUT2 _307_ (INIT=4'0100)   I0=_214_  I1=_221_
```

`INIT=4'0100`: bit[2]=1 → `{I1,I0}=10` → `_213_ = _221_ & ~_214_`. `_221_` is 1 when the state counter is at 11 and `io_inputDone` is rising — the exact cycle when the hardcoded 0x0A byte (byte 4 of each round) is strobed. `_214_` is a 6-input NOR that evaluates to 0 under normal active-input conditions, leaving `_213_ = _221_`.

**Round 1 result:** send bytes 'H' (72), 'T' (84), 'B' (66) as bytes 1–3. The harness appends 0x0A automatically as byte 4. At state=11 on that strobe, `_264_=1` and `_213_=1`, so `_193_=1` and `_211d3dff[4]` latches high for the rest of the run.

**Chain dependency:** bit[4] is the starting link because its only prerequisite beyond the input registers is `_211d3dff[0]` (the FDPE preset bit, permanently 1 after reset). For each subsequent bit, the corresponding comparison LUT tree contains `_211d3dff[k]` for the previous stage as a required input — visible at `CtfTask.v:2407` where `_211d3dff[4]` is wired into the arming logic for bit[2]. The same pattern repeats down the chain: bit[2]→bit[6]→bit[1]→bit[5]→bit[3]→bit[7]. Sending rounds out of order leaves the prerequisite bit unset and the later stage never fires.

## Step 6 — Verify and pull the flag

The unlocking input is the flag itself: **`HTB{BCKD000RD_HRDWR3}`** — 21 characters, seven rounds of three ASCII characters each. Decoding the LUT INIT constants and MUX trees for each prerequisite bit yields the exact required values:

| Round | Sets bit | Bytes sent (ASCII) | Input chars |
|-------|----------|--------------------|-------------|
| 1     | [4]      | H (72), T (84), B (66)   | `HTB` |
| 2     | [2]      | { (123), B (66), C (67)  | `{BC` |
| 3     | [6]      | K (75), D (68), 0 (48)   | `KD0` |
| 4     | [1]      | 0 (48), 0 (48), R (82)   | `00R` |
| 5     | [5]      | D (68), _ (95), H (72)   | `D_H` |
| 6     | [3]      | R (82), D (68), W (87)   | `RDW` |
| 7     | [7]      | R (82), 3 (51), } (125)  | `R3}` |

Each round also sends the hardcoded `0x0A` byte as byte 4, which strobes `io_inputDone` at state=11 and advances the state counter. After all seven rounds, `_211d3dff[7]` is latched high and the shutdown is disabled.

Pipe the candidate input into the binary:

```
$ echo "<candidate>" | ./VCtfTask
...
[8] Testing Emergency Shutdown:
[8] Input: '255', Power Level: '255'
[8] EMERGENCY SHUTDOWN MISSED!
TEST FAILED, EMERGENCY SHUTDOWN WAS NOT ACTIVED!
INPUT '<candidate>' NEEDS FURTHER ANALYSIS
```

The flag: **`HTB{BCKD000RD_HRDWR3}`**.

## Lessons

- **Hash-shaped signal names in synthesized RTL are intentional obfuscation.** Yosys preserves source identifiers by default. If a netlist has registers named `_<64 hex chars>_[i]`, somebody stripped the design to make it hard to audit.
- **Trace every input to a safety-critical signal.** The shutoff `_270_` should depend only on the temperature path. The instant you find it ANDed with anything else, that "anything else" is a candidate backdoor — even if it looks like an innocent enable signal.
- **A self-feedback flip-flop in a safety path is the signature.** A register bit whose D input is a LUT of (its own Q) and (user-controllable signals) is a one-shot kill-switch by construction. The trigger may be hard to reach, but the structure is enough to declare the design compromised.
