# High-Performance IEEE-754-Style FPU for RV32IF

This project implements and optimizes a single-precision floating-point unit for integration into a custom RV32IF RISC-V processor. The FPU supports addition/subtraction, multiplication, division, and square root using IEEE-754-style binary32 datapaths with guard/round/sticky rounding logic and major special-case handling.

The main goal is to reduce floating-point operation latency on FPGA by replacing sequential arithmetic algorithms with hardware-friendly parallel approaches. The divider was redesigned using Goldschmidt iterative division, the square-root unit was redesigned using fixed-point reciprocal Newton-Raphson refinement, and the adder was optimized using barrel-shift alignment and leading-zero-count normalization.

All modules were simulated with SystemVerilog testbenches and implemented in Xilinx Vivado on an Artix-7 target to evaluate latency, resource utilization, DSP usage, and timing closure.

---

## My Contributions

- Redesigned the divider using Goldschmidt iterative division with a 256-entry reciprocal LUT initial approximation.
- Implemented a fixed-point reciprocal square-root unit using Newton-Raphson refinement in Q24.24 arithmetic with a 512-entry LUT.
- Optimized the adder alignment and normalization path using a barrel shifter and combinational leading-zero counter.
- Built directed and randomized SystemVerilog testbenches for FADD/FSUB, FMUL, FDIV, and FSQRT covering normal operations, special cases, rounding boundaries, and latency behavior.
- Synthesized and implemented all optimized modules on Artix-7 using Vivado and analyzed post-implementation utilization and timing results.

> Some baseline FPU modules were initially derived from open-source educational implementations, including Dawson’s IEEE-754 FPU, and were significantly modified, optimized, or replaced. See individual module headers for details.

---

## Highlights

- **FSQRT:** latency reduced from ~550 cycles to ~18 cycles using fixed-point reciprocal Newton-Raphson refinement.
- **FDIV:** latency reduced from ~115 cycles to ~27 cycles using Goldschmidt division with parallel hardware multipliers.
- **FADD/FSUB:** variable-latency alignment and normalization loops replaced with a barrel shifter and leading-zero counter, achieving fixed 8-cycle latency for normal inputs.
- **FPGA implementation:** post-implementation Vivado results collected on Artix-7 `xc7a35tcsg324-1`.
- **Verification:** directed and randomized SystemVerilog testbenches covering normal arithmetic, NaN/Inf/zero cases, signed zeros, rounding boundaries, overflow/underflow, subnormal inputs, and latency behavior.
- **Dual-DUT comparison:** optimized divider and multiplier versions were tested against baseline implementations to compare both correctness and cycle count.

---

## Status

- RTL modules implemented: FADD/FSUB, FMUL, FDIV, and FSQRT.
- Directed SystemVerilog testbenches are available for all arithmetic units.
- Randomized testing is available for divider and multiplier.
- Post-implementation FPGA results were collected using Vivado on Artix-7.
- Integrated with the FloatCore-RV32IF processor; broader processor-level regression is ongoing.

---

## Optimization Summary

| Unit | Baseline Algorithm | Optimized Algorithm | Latency |
|---|---|---|---:|
| FADD/FSUB | Iterative align + normalize loops | Barrel shifter + LZC | 8 cycles fixed |
| FDIV | Restoring long division | Goldschmidt iterative division | ~115 → ~27 cycles |
| FSQRT | FP-level Newton-Raphson via FPU submodules | Fixed-point reciprocal Newton-Raphson | ~550 → ~18 cycles |
| FMUL | Direct 24×24 mantissa multiply | Interface/control cleanup | ~10–12 cycles |

---

## Architecture

```text
FPU-RTL/
├── adder.v / adder.sv      # FADD/FSUB — barrel shift + LZC normalization
├── divider.v               # Baseline restoring divider, reference version
├── divider.sv              # Goldschmidt divider, optimized version
├── multiplier.v            # FMUL — direct 24×24 mantissa multiply
├── sqrt.sv                 # FSQRT — fixed-point reciprocal Newton-Raphson
├── reciprocal_lut.hex      # 256-entry LUT for Goldschmidt initial seed
├── rsqrt_lut_512x24.hex    # 512-entry LUT for FSQRT initial seed
└── tb_*.sv                 # Testbenches for each module
```

Each module uses IEEE-754 binary32 encoding:

- 1 sign bit
- 8 exponent bits with bias 127
- 23 mantissa bits with an implicit leading 1 for normal numbers

The arithmetic datapaths use guard/round/sticky logic targeting Round-to-Nearest-Even behavior.

---

## Floating-Point Square Root

The optimized square-root unit computes the reciprocal square root first, then multiplies by the original operand:

```text
sqrt(x) = x × (1 / sqrt(x))
```

The reciprocal square-root approximation is refined using Newton-Raphson iteration in Q24.24 fixed-point arithmetic. This avoids repeated floating-point unpacking, normalization, and submodule handshaking.

### Key changes

- Replaced FP-level Newton-Raphson using divider/adder/multiplier submodules.
- Implemented a self-contained fixed-point reciprocal square-root datapath.
- Used a 512-entry LUT for the initial approximation.
- Reduced iteration count from 4 FP-level iterations to 2 fixed-point iterations.

### Result

```text
~550 cycles → ~18 cycles
≈ 30× latency reduction
```

---

## Floating-Point Divider

The optimized divider replaces restoring long division with Goldschmidt iterative division.

Restoring division produces quotient bits sequentially, which leads to high latency. Goldschmidt division instead refines the numerator and denominator using multiplicative correction factors, allowing FPGA DSP blocks to accelerate the operation.

### Key changes

- Replaced 50-step restoring long division.
- Added a 256-entry reciprocal LUT for the initial divisor approximation.
- Used parallel multipliers to update numerator and denominator.
- Kept additional iterations as a conservative precision margin.

### Result

```text
~115 cycles → ~27 cycles
≈ 4.3× latency reduction
```

---

## Floating-Point Adder / Subtractor

The adder/subtractor was optimized by removing variable-latency alignment and normalization loops.

### Key changes

- Replaced iterative exponent alignment with a barrel shifter.
- Replaced iterative normalization with a combinational leading-zero counter.
- Improved sticky-bit handling for rounding.
- Achieved fixed latency for normal input cases.

### Result

```text
Variable latency up to ~286 cycles → fixed 8 cycles for normal inputs
```

---

## FPGA Results — Post-Implementation

The following results are from Vivado post-implementation reports targeting the Arty A7-35 / Artix-7 `xc7a35tcsg324-1`.

| Module | LUTs | Registers | DSPs | WNS (ns) | Cycles |
|---|---:|---:|---:|---:|---:|
| Adder baseline | 328 | 388 | 0 | 4.240 | up to ~286 |
| Adder optimized | 775 | 291 | 0 | 1.340 | 8 fixed |
| Divider restoring | 415 | 402 | 0 | 1.113 | ~115 |
| Divider Goldschmidt | 616 | 302 | 18 | 0.213 | ~27 |
| FSQRT NR via FPU | 1178 | 1171 | 2 | 10.679 | ~550 |
| FSQRT fixed-point NR | 751 | 132 | 35 | 0.488 | ~18 |

Clock targets:

- FSQRT experiments: 18 ns period
- Other module experiments: 16 ns period

All listed modules achieved timing closure under their stated constraints.

---

## Verification

FPU modules were verified using dedicated SystemVerilog testbenches. Verification covers both functional correctness and cycle-count behavior.

| Module | Test Cases | Tolerance | Key Feature |
|---|---:|---|---|
| FADD/FSUB | 30 directed | ±2 ULP | Internal state monitoring per cycle |
| FMUL | 39 directed + 1000 random | ±2 ULP | Dual-DUT comparison vs baseline |
| FDIV | 39 directed + 1000 random | ±2 ULP | Dual-DUT: Goldschmidt vs restoring |
| FSQRT | 24 directed | Defined relative-error tolerance | Q24.24 debug ports and `$sqrt()` reference |

Verification includes:

- Normal arithmetic operations
- NaN, infinity, and zero special cases
- Signed-zero behavior
- Rounding-boundary cases
- Overflow and underflow cases
- Subnormal input cases
- Cycle-count validation
- FSM deadlock detection using timeout watchdogs

Current verification uses defined ULP and relative-error tolerances depending on the module. Larger randomized ULP-based regression using a SoftFloat-style reference model is planned for stricter correctness validation.

---

## How to Run

### Tools required

- QuestaSim or ModelSim
- Xilinx Vivado for synthesis / implementation

### Simulate adder testbench

```tcl
vlib work
vlog adder.sv tb_adder_fast.sv
vsim -do "run -all" work.tb_adder_fast
```

### Simulate divider dual-DUT comparison

```tcl
vlib work
vlog divider.v divider.sv goldschmidt_divider_tb.sv
vsim -do "run -all" work.goldschmidt_divider_tb
```

### Simulate square-root testbench

```tcl
vlib work
vlog sqrt.sv tb_fsqrt_ultrafast.sv
vsim -do "run -all" work.tb_fsqrt_ultrafast
```

### FPGA implementation

Target device:

```text
xc7a35tcsg324-1
```

Recommended flow:

1. Open Vivado.
2. Create a project targeting Arty A7-35 / Artix-7 `xc7a35tcsg324-1`.
3. Add the selected FPU RTL modules.
4. Add clock constraints.
5. Run synthesis and implementation.
6. Check utilization and timing reports.

---

## Limitations and Future Work

- This is an IEEE-754-style binary32 FPU, not a fully standards-complete IEEE-754 implementation.
- IEEE-754 exception flags such as overflow, underflow, invalid, divide-by-zero, and inexact are not currently propagated to architectural CSRs.
- FSQRT verification currently uses directed cases and a defined relative-error tolerance; larger ULP-based randomized validation is planned.
- The Goldschmidt divider uses a conservative iteration count; reducing iterations after formal precision/error analysis is future work.
- Larger reference-model-based testing using a SoftFloat-style or Python golden model is planned.
- Further integration testing inside the full RV32IF pipeline is ongoing.

---


# Full Technical Report

Detailed per-module analysis, mathematical derivations, algorithm comparisons, cycle-count breakdowns, verification discussion, and complete FPGA implementation data are documented in:

[`docs/FPU_Technical_Report.pdf`](docs/FPU_Technical_Report.pdf)

## Related Project

This FPU is integrated into a complete 5-stage RV32IF pipelined processor with UVM verification:

[FloatCore-RV32IF](https://github.com/mo1234-bit/RISC-V)

---

## Keywords

`RISC-V` `RV32IF` `FPU` `Floating-Point Unit` `IEEE-754` `SystemVerilog` `Verilog` `FPGA` `Vivado` `Artix-7` `Goldschmidt Division` `Newton-Raphson` `UVM` `Computer Architecture` `Digital IC Design`
