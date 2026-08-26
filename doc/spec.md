# FP32 Multiplier Specification

## 1. Overview

Implement a multi-cycle single-precision floating-point multiplier.

The module performs:

    z = a * b

where `a`, `b`, and `z` are IEEE-754 binary-32 values.

The implementation must be synthesizable.

The design accepts one operation at a time and uses a valid/busy/out_valid
handshake.

The primary correctness criteria are:

- Correct results for normal FP32 operands.
- Correct handling of signed zero.
- Correct overflow to infinity.
- Correct treatment of results at the smallest normal exponent.
- Exact 7-cycle latency and correct handshake.

---

# 2. Module Interface

Module name:

    fmultiplier

Ports:

| Port       | Direction | Width | Description |
|------------|-----------|-------|-------------|
| clk        | input     | 1     | Clock |
| rst        | input     | 1     | Asynchronous active-high reset |
| valid      | input     | 1     | One-cycle operation request |
| a          | input     | 32    | FP32 operand A |
| b          | input     | 32    | FP32 operand B |
| z          | output    | 32    | FP32 result |
| out_valid  | output    | 1     | One-cycle result-valid pulse |

Do not change the module name or any port.

---

# 3. Reset Behavior

`rst` is asynchronous and active-high.

When `rst == 1`:

- busy must be cleared.
- out_valid must be cleared.
- z must be reset to a defined value (e.g., 0).
- internal state must be reset.
- no operation may be considered active.

After reset is released, the module must be ready to accept a new operation.

---

# 4. Handshake

An operation is accepted only when:

    busy == 0
    valid == 1

On the rising edge where the operation is accepted:

- capture `a` into an internal register,
- capture `b` into an internal register,
- assert busy,
- begin the multi-cycle operation.

Once the operation starts:

- changes to external `a` and `b` must not affect the current operation.
- additional `valid` pulses while busy must be ignored.
- the current operation must continue until completion.

---

# 5. Latency

The multiplier has a fixed latency of 7 clock cycles.

For every accepted operation:

    valid accepted
          |
          | 7 clock cycles
          v
    out_valid == 1

`out_valid` must be asserted for exactly one clock cycle.

When `out_valid == 1`, `z` must contain the result corresponding to the
accepted operands.

---

# 6. FP32 Representation

The 32-bit FP32 representation is:

    [31]      Sign
    [30:23]   Exponent
    [22:0]    Fraction

For normal numbers:

    value = (-1)^sign × 1.fraction × 2^(exponent - 127)

The hidden leading bit must be included when constructing the mantissa.

For a normal operand:

    mantissa = {1'b1, fraction}

Therefore the mantissa width is 24 bits.

---

# 7. Sign Calculation

The result sign is:

    result_sign = sign_a XOR sign_b

This applies to normal operands and signed zero.

---

# 8. Exponent Calculation

Extract the unbiased exponent from each normal operand:

    a_e = exponent_a - 127
    b_e = exponent_b - 127

The multiplier must calculate the intermediate exponent exactly as follows:

    z_e = a_e + b_e

The implementation must NOT add an additional bias adjustment at this stage.

Normalization may modify `z_e` when required by the mantissa product.

The final FP32 biased exponent is:

    biased_exponent = z_e + 127

---

# 9. Mantissa Multiplication

For normal operands:

    a_m = {1'b1, fraction_a}
    b_m = {1'b1, fraction_b}

Multiply:

    product = a_m * b_m

The full multiplication result must be retained with sufficient width.
`a_m` and `b_m` are 24 bits each, so `product` requires 48 bits.

Do not truncate the multiplication before normalization/rounding.

---

# 10. Normalization

The product of two normalized 24-bit mantissas is in the range:

    [1.0, 4.0)

Therefore:

- if the product is >= 2.0, shift the mantissa right by one bit and increment
  the exponent by one.
- otherwise retain the mantissa position.

The normalization decision must be based on the actual product bits.

---

# 11. Rounding

The result must use round-to-nearest-even when rounding is required.

The implementation must retain sufficient guard/round/sticky information
before truncating the mantissa.

If rounding causes the mantissa to overflow:

    1.xxxxx -> 10.000...

then normalize again and increment the exponent.

---

# 12. Overflow

If the final exponent exceeds the maximum representable normal FP32 exponent:

    result = infinity

The result must preserve the calculated sign.

For positive overflow:

    {1'b0, 8'hFF, 23'h000000}

For negative overflow:

    {1'b1, 8'hFF, 23'h000000}

---

# 13. Underflow and Denormal Boundary

The smallest normal unbiased exponent is:

    -126

Therefore:

    z_e == -126

is still a normal FP32 result.

Only:

    z_e < -126

enters the underflow/denormal handling path.

A result sitting exactly at `z_e == -126` must be packed as a normal number,
with biased exponent field `1` and its computed fraction bits. Forcing the
exponent field to zero for every result at or below -126 silently destroys
these valid normal results.

For `z_e < -126`, the implementation should:

- shift the mantissa right as needed,
- and produce either a subnormal or zero according to IEEE-754 rules.

Exact bit-perfect subnormal behavior is desirable but not the primary focus;
the key requirement is correct handling at the `z_e == -126` boundary.

---

# 14. Zero Handling

If either operand is zero:

    result = signed zero

The result sign is:

    sign_a XOR sign_b

Examples:

    +0 * +1 = +0
    -0 * +1 = -0
    +0 * -1 = -0
    -0 * -1 = +0

Zero operands must be correctly recognized even if other special-value
handling is simplified.

---

# 15. Special Values

Unless explicitly required elsewhere in this specification:

- NaN handling is not required.
- Infinity input handling is not required.
- Subnormal input handling is not required.

Operands are expected to be normal FP32 values with biased exponent in the
range 1..254, plus signed zero.

The implementation must not invent additional behavior that is not specified.

---

# 16. Required Test Cases

The verification environment must cover at least the following. These are the
behaviors that determine correctness for this design — cover them well rather
than maximizing the number of test cases.

### Basic magnitudes

    1.0 * 1.0 = 1.0
    2.0 * 2.0 = 4.0
    2.0 * 3.0 = 6.0
    4.0 * 4.0 = 16.0
    1.5 * 2.0 = 3.0
    5.0 * 0.5 = 2.5

### Fractional / rounding

    0.5 * 0.5 = 0.25
    0.75 * 0.75 = 0.5625
    0.25 * 0.25 = 0.0625

Include at least one operand pair whose exact product needs more than 24
mantissa bits, so round-to-nearest-even is actually exercised.

### Signs

    (-1.0) * 1.0 = -1.0
    (-1.0) * (-1.0) = 1.0
    (-2.0) * 3.0 = -6.0

### Boundary

    largest normal * 1.0
    an overflow-producing multiplication
    a product landing at the smallest normal exponent (z_e == -126)

### Handshake

    result appears exactly 7 cycles after an accepted valid
    out_valid asserted for exactly one cycle
    valid ignored while busy

---

# 17. Reference Values

Expected FP32 values MUST be independently verified.

Expected values must not be changed to match an incorrect RTL implementation.

---

# 18. Verification Requirement

The implementation is considered correct only when:

- the RTL compiles,
- all required tests pass,
- latency is exactly 7 cycles,
- handshake behavior is correct,
- result bits match the expected FP32 values,
- no module ports have been changed.

A partial pass is not considered completion.