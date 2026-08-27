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

The primary correctness criteria are:

- Correct results for normal FP32 operands.
- Correct handling of signed zero.
- Correct overflow to infinity.
- Correct treatment of results at the smallest normal exponent (z_e = −126).
- Exact 7-cycle latency and correct handshake.

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
- `product`: at least 48-bit product of mantissas (50-bit is acceptable)  
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

- Extract fields from `a_r` and `b_r`:
  - `a_s = a_r[31]`, `b_s = b_r[31]`
  - `expA = a_r[30:23]`, `expB = b_r[30:23]`
  - `fracA = a_r[22:0]`, `fracB = b_r[22:0]`
- Convert biased exponents into unbiased form:
  - `a_e = expA - 127`
  - `b_e = expB - 127`
- Initialize mantissas:
  - `a_m = {1'b0, fracA}`
  - `b_m = {1'b0, fracB}`

### Stage 2 — Special classification + denormal setup

- Classify operands using flags derived from `a_r`/`b_r`:
  - `a_is_zero`, `b_is_zero`
  - `a_is_inf`, `b_is_inf` (optional structure only; inputs are expected normal)
  - `a_is_nan`, `b_is_nan` (optional structure only)
- Classification rules:
  - Zero: `exp == 0` and `mant == 0`.
  - Infinity: `exp == 255` and `mant == 0`.
  - NaN: `exp == 255` and `mant != 0`.
- For **normal operands** (primary path):
  - If exponent ≠ 0 ⇒ set implicit leading 1:
    - `a_m[23] = 1`, `b_m[23] = 1`.
- For **subnormal operands** (not required for primary tests, but structure may exist):
  - If exponent = 0 and mantissa ≠ 0 ⇒ treat as subnormal:
    - Force exponent to −126 (subnormal baseline).
    - Do not set implicit leading 1.

> For the primary test mode:
> - Operands are normal with `exp ∈ [1..254]`.
> - Hidden-one insertion always happens.
> - Special-case logic is structurally present but rarely exercised.

### Stage 3 — Input normalization (lightweight)

- If mantissa MSB is not set:
  - Left-shift mantissa until MSB is 1 or exponent reaches −126.
  - Decrement exponent for each left shift.
- This ensures that both normal and (optionally) subnormal inputs are in a consistent normalized form before multiplication.
- For strictly normal inputs with hidden 1 already set, this stage typically does nothing.

### Stage 4 — Multiply core

- Compute result sign:  
  \[
  z\_s = a\_s \oplus b\_s
  \]
- Exponent add (no extra bias here):  
  \[
  z\_e = a\_e + b\_e
  \]
- Mantissa product:
  - Use 24-bit mantissas:
    - `a_m = {1'b1, fracA}` for normals
    - `b_m = {1'b1, fracB}` for normals
  - Compute:
    \[
    \text{product} = a\_m \times b\_m
    \]
  - `product` must be at least 48 bits wide; 50 bits is acceptable.
- Do **not** add any extra bias adjustment in this stage beyond `z_e = a_e + b_e`.

> Note: Some implementations scale the product (e.g., `*4`) and adjust later;
> if you do so, ensure the net effect matches the standard FP multiplication
> semantics and the normalization rules in Stage 6.

### Stage 5 — Extract mantissa + rounding bits

- From the full product:
  - Choose a 24-bit (or wider) intermediate mantissa window and rounding bits.
  - A common pattern:
    - `z_m = product[49:26]` (24 bits)
    - `guard_bit = product[25]`
    - `round_bit = product[24]`
    - `sticky = OR(product[23:0])`
- The exact bit indices may vary if you use a different internal scaling, but:
  - You must retain enough bits to implement correct RNE.
  - The combination `{z_m, guard_bit, round_bit, sticky}` must fully represent the rounded result.

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)

This stage performs three logically sequential sub-steps:

1. **Normalization of the product mantissa**
2. **Underflow alignment** toward exponent −126
3. **RNE rounding**

These sub-steps are **sequentially dependent**:

- Normalization must see the initial product mantissa.
- Underflow alignment must see the result of normalization.
- Rounding must see the result of underflow alignment.

**Implementation note (critical):**  
If you implement these sub-steps using multiple `if` blocks with non-blocking assignments (`<=`) on the same signals (`z_m`, `guard_bit`, `round_bit`, `sticky`, `z_e`), each block will read the **pre-cycle** register value, not the updated value from the previous sub-step. This silently produces wrong results whenever more than one sub-step is needed for the same operand.

A reliable pattern:

- Declare local variables at the top of this stage, e.g.:
  - `logic [49:0] temp_product;`
  - `logic [23:0] temp_mantissa;`
  - `logic temp_guard, temp_round, temp_sticky;`
  - `logic signed [9:0] temp_exp;`
- Use **blocking assignments (`=`)** to carry values through:
  - normalization → underflow alignment → rounding.
- Use a **single non-blocking assignment (`<=`)** at the end of the stage to commit:
  - `z_m <= temp_mantissa;`
  - `guard_bit <= temp_guard;`
  - `round_bit <= temp_round;`
  - `sticky <= temp_sticky;`
  - `z_e <= temp_exp;`

#### 6.1 Normalization

- The product of two 24-bit mantissas is in the range `[1.0, 4.0)`.
- If the product is ≥ 2.0:
  - Shift the mantissa right by 1 bit.
  - Increment `z_e` by 1.
- Otherwise, keep the mantissa position unchanged.

#### 6.2 Underflow alignment

- The smallest normal unbiased exponent is:
  - `-126`
- If `z_e < -126`:
  - Compute shift amount:
    \[
    sh = -126 - z\_e
    \]
  - Shift the mantissa right by `sh`.
  - Accumulate shifted-out bits into `sticky`.
  - Set `z_e = -126`.
- If `z_e >= -126`, no underflow alignment is needed.

#### 6.3 RNE rounding

- After normalization and underflow alignment:
  - Let `LSB = z_m[0]`.
  - Round up if:
    \[
    guard\_bit = 1 \ \text{and}\ (round\_bit \lor sticky \lor LSB) = 1
    \]
- If rounding up:
  - `z_m = z_m + 1`.
  - If this causes mantissa overflow (e.g., `24'b111...1 + 1` → `24'b1_000...0`):
    - Set `z_m` to `24'd1 << 23` (i.e., `1.0` in fixed-point form).
    - Increment `z_e` by 1.

---

## Stage 7 — Pack

- Compute biased exponent:
  \[
  \text{expZ} = z\_e + 127
  \]

- **Special-case priority** (evaluate before normal packing):

  1. **Zero cases**:
     - If either operand is zero and no infinity is involved:
       - Result is ±0 with `z_s = a_s ^ b_s`.
       - Output: `{z_s, 31'd0}`.

  2. **Overflow**:
     - If `expZ >= 255`:
       - Output ±Inf with `z_s`:
         - `{z_s, 8'd255, 23'd0}`.

  3. **Normal vs denormal boundary**:
     - If `z_e == -126` (i.e., `expZ == 1`):
       - This is still a **normal** number.
       - Pack as:
         - `exp = 1`
         - `fraction = z_m[22:0]`.
     - Do **not** force the exponent field to 0 for results at `z_e == -126`.

  4. **Underflow to subnormal/zero**:
     - If `z_e < -126` after alignment:
       - Produce either a subnormal or zero according to IEEE-754 rules.
       - Exact bit-perfect subnormal behavior is desirable but secondary to:
         - correct handling at `z_e == -126`,
         - correct overflow and zero behavior.

- **Normal packing** (when no special case above applies and `1 <= expZ <= 254`):
  - `z[31] = z_s`
  - `z[30:23] = expZ[7:0]`
  - `z[22:0] = z_m[22:0]`

- Assert `out_valid` for one cycle and clear `busy`.

---

## Assumptions & Constraints

- Primary operating mode:
  - Inputs are **normal FP32 values** with biased exponent in `[1..254]`, plus signed zero.
  - NaN/Inf/subnormal **inputs** are not required for primary correctness.
- The implementation must:
  - Correctly handle normal × normal → normal/inf/zero/subnormal as per IEEE-754.
  - Correctly handle:
    - Zero × finite → zero (with correct sign).
    - Overflow → infinity (with correct sign).
    - Results at `z_e == -126` as normal numbers.
- Minor deviations in rare subnormal corner cases are acceptable, but the design must not lock up or produce completely incorrect encodings.

---

## Code Style and Documentation Requirements

The generated RTL (`multiply_fp32.sv`) must satisfy:

- **Top module header comment block** including:
  - Brief description of the module.
  - Latency (7 cycles) and throughput (1 per 7 cycles).
  - Supported input domain (normals with correct zero/overflow/−126 boundary behavior).
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
- **No SystemVerilog Assertion (SVA) property/sequence syntax**.

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

## Verification Expectations

- The design should produce correct IEEE-754 results for:
  - Normal × normal operands across a wide range of exponent and mantissa values.
  - Combinations involving:
    - ±0
    - Overflow to ±Inf
    - Results at the smallest normal exponent (`z_e == -126`).
- Subnormal outputs may occur for very small results; approximate but sensible behavior is acceptable as long as:
  - The `z_e == -126` boundary is correct.
  - Overflow and zero behavior are correct.
- Minor deviations in rare subnormal corner cases are acceptable.

---

## File Naming

- Top-level RTL file: **`multiply_fp32.sv`**
- Top module name: **`fmultiplier`**

These names must be used exactly to match the integration environment.