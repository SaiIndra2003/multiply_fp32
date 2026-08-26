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

The implementation must correctly handle normal operands and must also produce correct results for common special cases (zeros, infinities, and basic NaN behavior) to improve overall correctness across a wide range of inputs.

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
  - `a_is_nan`, `b_is_nan`
- Classification rules:
  - Zero: `exp == 0` and `mant == 0`.
  - Infinity: `exp == 255` and `mant == 0`.
  - NaN: `exp == 255` and `mant != 0`.
- For **normal operation**:
  - If exponent ≠ 0 ⇒ set implicit leading 1: `a_m[23] = 1`, `b_m[23] = 1`.
- For **subnormal inputs**:
  - If exponent = 0 and mantissa ≠ 0 ⇒ treat as subnormal:
    - Force exponent to −126 (subnormal baseline).
    - Do not set implicit leading 1; use mantissa as-is (with leading zeros).

### Stage 3 — Input normalization (lightweight)

- If mantissa MSB is not set:
  - Left-shift mantissa until MSB is 1 or exponent reaches −126.
  - Decrement exponent for each left shift.
- This ensures that both normal and subnormal inputs are in a consistent normalized form before multiplication.

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
  \text{product} = a\_m \times b\_m \times 4
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
   - If the resulting exponent is below −126 after alignment, the value will be represented as a subnormal or zero in Stage 7.

2. **Normalization** if MSB missing:
   - Left-shift mantissa until MSB is 1 or exponent reaches −126.
   - Increment exponent for each left shift.
   - Carry guard into LSB as needed during shifts.

3. **RNE rounding**:
   - Round up if:  
     \[
     G = 1 \ \text{and}\ (R \lor S \lor \text{LSB}) = 1
     \]
   - If rounding overflows mantissa:
     - Set mantissa to `0x800000` (i.e., 1.0 in fixed-point form).
     - Increment exponent.

### Stage 7 — Pack

- Compute biased exponent:  
  \[
  \text{expZ} = z\_e + 127
  \]

- **Special-case priority** (must be evaluated before normal packing):

  1. **NaN propagation**:
     - If either input is NaN:
       - Output a NaN with:
         - `exp = 255`
         - `mant != 0` (at least one bit set; MSB of mantissa preferably 1 for quiet NaN).
       - Sign can be copied from one of the NaN inputs or set to 0; either is acceptable as long as the result is a valid NaN.

  2. **Infinity cases**:
     - If either input is ±Inf and the other is:
       - Non-zero finite ⇒ result is ±Inf with `z_s = a_s ^ b_s`.
       - Zero ⇒ result is NaN (invalid operation ∞ × 0).
     - Implement:
       - If `(a_is_inf && !b_is_zero) || (b_is_inf && !a_is_zero)` ⇒ output ±Inf.
       - If `(a_is_inf && b_is_zero) || (b_is_inf && a_is_zero)` ⇒ output NaN.

  3. **Zero cases**:
     - If either input is zero and no infinity is involved:
       - Result is ±0 with `z_s = a_s ^ b_s`.

- **Normal/subnormal packing** (when no special case above applies):

  - If `expZ >= 255`:
    - Overflow ⇒ output ±Inf with `z_s`.
  - Else if `expZ <= 0`:
    - Underflow to subnormal/zero:
      - If the normalized mantissa cannot be represented with `exp = 0`, output ±0.
      - Otherwise, output a subnormal:
        - `exp = 0`
        - Fraction = appropriate bits of mantissa (after shifting for denormal representation).
  - Else:
    - Normal number:
      - `z[31] = z_s`
      - `z[30:23] = expZ[7:0]`
      - `z[22:0] = z_m[22:0]`

- Assert `out_valid` for one cycle and clear `busy`.

---

## Assumptions & Constraints

- The primary functional target is correct behavior for:
  - Normal × normal → normal/inf/zero/subnormal as per IEEE-754.
  - Common special-case combinations:
    - Zero × finite → zero
    - Finite × zero → zero
    - Inf × non-zero finite → inf
    - Inf × zero → NaN
    - NaN × anything → NaN
- Subnormal inputs and outputs must be handled according to the rules above; the design should not lock up or produce completely incorrect encodings for these cases.
- The implementation should be robust for all 32-bit input patterns, even if some corner cases are not perfectly bit-exact.

---

## Code Style and Documentation Requirements

The generated RTL (`multiply_fp32.sv`) must satisfy:

- **Top module header comment block** including:
  - Brief description of the module.
  - Latency (7 cycles) and throughput (1 per 7 cycles).
  - Supported input domain (normals with full special-case handling for zero, inf, NaN, and subnormals).
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

## Configurability

Add a parameter to allow future extension:

```systemverilog
parameter STRICT_NORMAL_ONLY = 1;
```

- When `STRICT_NORMAL_ONLY == 1`:
  - The design may assume inputs are primarily normal but must still implement:
    - Zero handling
    - Infinity handling
    - NaN propagation
    - Subnormal output handling
- When `STRICT_NORMAL_ONLY == 0`:
  - Full special-case classification and handling is expected, with more precise subnormal and NaN behavior.

The parameter must exist and be used in `if` conditions around optional optimizations, but all required special-case behavior must be present regardless of its value.

---

## Verification Expectations

- The design should produce correct IEEE-754 results for:
  - Normal × normal operands across a wide range of exponent and mantissa values.
  - Combinations involving:
    - ±0
    - ±Inf
    - NaN
    - Subnormal inputs and outputs (where representable).
- Minor deviations in rare corner cases (e.g., specific NaN payload propagation) are acceptable, but the overall behavior must be consistent with IEEE-754 rules for sign, exponent, and special-case outcomes.

---

## File Naming

- Top-level RTL file: **`multiply_fp32.sv`**
- Top module name: **`fmultiplier`**

These names must be used exactly to match the integration environment.