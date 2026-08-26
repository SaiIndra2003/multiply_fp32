# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview

`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

This design targets:

- **Bit-accurate results for normal FP32 numbers** (IEEE-754 round-to-nearest-even),
- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
- Functional behavior:  
  \[
  z = a \times b
  \]
  where `z`, `a`, and `b` are 32-bit IEEE-754 single-precision numbers.

The primary verification will use **normal operands only** (exp ∈ [1..254]) and will skip cases where the mathematically correct result is subnormal. The RTL must still be structurally ready to detect zero/inf encodings, but full IEEE-754 special-case coverage is not required for tests to pass.

---

## Interface

### Ports

| Port        | Dir | Width | Description                                      |
|------------|-----|-------|--------------------------------------------------|
| `clk`      | in  | 1     | Clock                                            |
| `rst`      | in  | 1     | Async reset (posedge)                            |
| `valid`    | in  | 1     | **1-cycle start pulse**; accepted only when idle |
| `a`        | in  | 32    | Operand A (FP32 bits)                            |
| `b`        | in  | 32    | Operand B (FP32 bits)                            |
| `z`        | out | 32    | Result (FP32 bits)                               |
| `out_valid`| out | 1     | **1-cycle pulse** when `z` is updated/valid      |

### Handshake contract

- When `busy == 0`, a high `valid` on a rising clock edge **starts** an operation:
  - `a` and `b` are **registered** into internal regs `a_r` and `b_r`.
  - The FSM begins at `counter = 1`.
- While `busy == 1`, new `valid` pulses are **ignored**.
- When the operation completes:
  - `z` is updated,
  - `out_valid` pulses high for 1 clock cycle,
  - `busy` is cleared.

The module may implement `busy` internally; it does not need to be a top-level port.

---

## Latency and Throughput

### Latency

- Fixed latency of **7 stages**.
- The operation begins at stage `counter = 1` and completes at `counter = 7`.
- `out_valid` asserts on the cycle where stage 7 packing finishes.

System-level expectation:

- **`out_valid` occurs 7 clock cycles after the start edge** (the edge where `valid` was sampled when idle).

### Throughput

- **Not pipelined** (single-issue).
- Maximum throughput: **1 result per 7 cycles** (assuming `valid` is asserted only when idle).

---

## Internal Data Model (IEEE-754 binary32)

For each operand:

- `sign` = bit 31  
- `exp`  = bits 30:23 (biased exponent, bias = 127)  
- `mant` = bits 22:0 (fraction)

Internal signals:

- `a_s, b_s, z_s`: sign bits  
- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, used with `$signed`)  
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable  
- `product`: 50-bit product of mantissas  
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for round-to-nearest-even (RNE)

---

## FSM / Pipeline Stages

The FSM is controlled by:

- `busy` (operation in progress)  
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential `always_ff` block using `case(counter)`.

The 7 stages must be **clearly visible in the code**, with comments indicating the stage name and purpose:

1. `// Stage 1 — UNPACK`
2. `// Stage 2 — SPECIAL_CLASSIFY`
3. `// Stage 3 — NORMALIZE_INPUTS`
4. `// Stage 4 — MUL_CORE`
5. `// Stage 5 — EXTRACT_ROUND_BITS`
6. `// Stage 6 — NORMALIZE_ROUND`
7. `// Stage 7 — PACK`

### Stage 1 — Unpack

- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form:  
  \[
  a\_e = \text{expA} - 127,\quad b\_e = \text{expB} - 127
  \]
- Capture signs `a_s`, `b_s`.

### Stage 2 — Special classification + denormal setup

- Classify operands using flags derived from `a_r`/`b_r`:
  - `a_is_zero`, `b_is_zero`
  - `a_is_inf`, `b_is_inf`
  - `a_is_nan`, `b_is_nan` (optional for this grade band)
- For **normal operation** (primary path used by tests):
  - If exponent ≠ 0 ⇒ set implicit leading 1: `a_m[23] = 1`, `b_m[23] = 1`.
- For **subnormal inputs** (not used in primary tests, but structure should exist):
  - If exponent = 0 ⇒ force exponent to −126 (subnormal baseline) and handle mantissa without implicit 1.

> For the primary test mode:
> - `expA` and `expB` are in [1..254],
> - hidden-one insertion always happens,
> - special-case logic is effectively bypassed except for structural presence.

### Stage 3 — Input normalization (lightweight)

- If mantissa MSB is not set, shift left and decrement exponent.
- Mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply core

- Compute result sign:  
  \[
  z\_s = a\_s \oplus b\_s
  \]
- Exponent add:  
  \[
  z\_e = a\_e + b\_e + 1
  \]
- Mantissa product:  
  \[
  \text{product} = a\_m \times b_m \times 4
  \]
  The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

### Stage 5 — Extract mantissa + rounding bits

- Mantissa bits:  
  \[
  z\_m = \text{product}[49:26]
  \]
- Rounding bits:
  - `guard_bit = product[25]`
  - `round_bit = product[24]`
  - `sticky = OR(product[23:0])`

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)

This stage performs:

1. **Underflow alignment** toward exponent −126:
   - If `z_e < -126`:
     - Compute shift amount:  
       \[
       sh = -126 - z\_e
       \]
     - Shift mantissa right by `sh`, accumulating shifted-out bits into `sticky`.

2. **Normalization** if MSB missing:
   - Left-shift mantissa while adjusting exponent, carrying guard into LSB as needed.

3. **RNE rounding**:
   - Round up if:  
     \[
     G = 1 \ \text{and}\ (R \lor S \lor \text{LSB}) = 1
     \]
   - If rounding overflows mantissa:
     - Set mantissa to `0x800000` (i.e., 1.0 in fixed-point form).
     - Increment exponent.

### Stage 7 — Pack

- For the **normal path** (used by tests):
  - Convert unbiased exponent back to biased:  
    \[
    \text{expZ} = z\_e + 127
    \]
  - Pack:
    - `z[31] = z_s`
    - `z[30:23] = expZ`
    - `z[22:0] = z_m[22:0]`
  - If exponent indicates **overflow** ⇒ output ±Inf with correct sign.
  - If exponent indicates exact denormal boundary ⇒ force exponent field to 0 (denormal/zero representation).

- **Special-case overrides** (structure must exist, but tests primarily use normals):
  - If either operand is zero:
    - Result = ±0 with `z_s = a_s ^ b_s`.
  - If one operand is ±Inf and the other is non-zero finite:
    - Result = ±Inf with `z_s = a_s ^ b_s`.
  - NaNs:
    - For this grade band, it is acceptable to treat NaNs as “unspecified” as long as the design does not lock up.

- Assert `out_valid` for one cycle and clear `busy`.

---

## Assumptions & Constraints

Primary operating mode (used by tests):

- Inputs: `exp ∈ [1..254]` (normal numbers).
- Testbench will:
  - Generate only normal operands by default.
  - Skip cases where the mathematically correct result is subnormal.

The RTL must still:

- Detect zero and infinity encodings and have structural logic for them.
- Not lock up for any 32-bit input pattern.

---

## Code Style and Documentation Requirements

The generated RTL (`multiply_fp32.sv`) must satisfy:

- **Top module header comment block** including:
  - Brief description of the module.
  - Latency (7 cycles) and throughput (1 per 7 cycles).
  - Supported input domain (normals, with structural zero/inf detection).
  - Author and date placeholders.
- **Clear FSM implementation**:
  - Use a `counter` (1..7) inside a single `always_ff` block.
  - Each `case(counter)` block must have a comment with the stage name and a 1–2 line description.
- **Consistent naming**:
  - `a_r`, `b_r` for registered inputs.
  - `a_s`, `b_s`, `z_s` for signs.
  - `a_e`, `b_e`, `z_e` for unbiased exponents.
  - `a_m`, `b_m`, `z_m` for 24-bit mantissas.
  - `guard_bit`, `round_bit`, `sticky` for rounding.
- **Synthesizable logic** for the main datapath and FSM.
- **No SystemVerilog Assertion (SVA) property/sequence syntax** (Icarus Verilog is used for simulation).

---

## Runtime Checks (Simulation Only)

The design may include **procedural checks** for simulation (non-synthesizable), guarded by `ifndef SYNTHESIS`, such as:

- Warning if `valid` is asserted while `busy`:
  ```systemverilog
  if (busy && valid)
      $display("%0t: WARNING: valid asserted while busy; ignored.", $time);
  ```
- Error if `out_valid` is asserted when `counter != 7`:
  ```systemverilog
  if (out_valid && (counter != 4'd7))
      $error("%0t: ERROR: out_valid asserted at counter=%0d", $time, counter);
  ```

These checks should not affect synthesis and must not use SVA syntax.

---

## Configurability

Add a parameter to allow future extension:

```systemverilog
parameter STRICT_NORMAL_ONLY = 1;
```

- When `STRICT_NORMAL_ONLY == 1`:
  - The design may assume inputs are primarily normal.
  - Special-case logic can be simplified but must still be structurally present.
- When `STRICT_NORMAL_ONLY == 0`:
  - Full special-case classification (zero, inf, NaN) is expected (can be left as a stub for this grade band, but structure must be present).

The parameter must exist and be used in `if` conditions around special-case logic.

---

## Verification Notes

Recommended testbench behavior:

- Drive `a`/`b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- For primary verification:
  - Use **normal operands** (exp ∈ [1..254]).
  - Skip cases where the mathematically correct result is subnormal.

The provided Python/cocotb testbench is compatible with this spec, with environment controls such as:

- `ALLOW_NAN` (default: 0)
- `ALLOW_INF` (default: 0)
- `SPECIAL_RATE` (fraction of special operands)

---

## File Naming

- Top-level RTL file: **`multiply_fp32.sv`**
- Top module name: **`fmultiplier`**

These names must be used exactly to match the test environment.