# Single-cycle Synthesis Record

## Historical v0.1 run

First run and closeout reproduction: 2026-09-15, WSL Ubuntu-24.04.
Top: `rv32i_core`. The six source files below are the hand-written core and its
five child modules; the full-adder exercise and Python memory models are not
part of this synthesized hierarchy.

## Tool and reproduction

Yosys: `0.33 (git sha1 2584903a060)`.
From the repository root in WSL:

```sh
mkdir -p reports/synthesis build/synthesis
yosys -Q -T -l reports/synthesis/day13-core.log -p '
  read_verilog -sv rtl/rv32i_core.sv rtl/pc.sv rtl/decoder.sv rtl/register_file.sv rtl/immediate_generator.sv rtl/alu.sv;
  synth -top rv32i_core;
  check -assert;
  stat;
  write_verilog -noattr build/synthesis/rv32i_core.v
'
```

The command was executed successfully with exit status 0. Log and netlist were
saved to the paths above. Both directories are ignored by Git; this tracked
record preserves the command and summary for future regeneration.
There is currently no `make synth-core` target.

Flow: parse synthesizable SystemVerilog, elaborate hierarchy, lower processes,
optimize and map memory/logic, map to generic gates, check the resulting design,
print statistics, then emit a generated Verilog netlist.

The flow preserves the module hierarchy; it does not use `synth -flatten`.
No target FPGA, Liberty library, SRAM macro, clock constraint, or physical
implementation was supplied. Default generic ABC mapping is not an ASIC PDK.

## Observed structural results

| Scope | Cells |
| --- | ---: |
| alu | 1483 |
| decoder | 93 |
| immediate_generator | 78 |
| pc | 143 |
| register_file | 3167 |
| rv32i_core, direct contents | 355, including five child instances |
| **Full hierarchy, primitive total** | **5314** |

Do not add all rows without removing the five counted child instances:
`1483 + 93 + 78 + 143 + 3167 + (355 - 5) = 5314`.

Additional observations:

- Register file: 1024 `$_DFFE_PP_` cells and 1984 `$_MUX_` cells.
- PC: 30 `$_SDFF_PP0_` plus 2 `$_SDFFE_PP0P_` cells: 32 flip-flops total.
- Entire hierarchy: 2420 MUX cells, 0 memory objects, and 0 procedural processes
  after synthesis. Storage is represented by flip-flops, not absent.
- `check -assert` reported `Found and reported 0 problems.`
- Combinational-process conversion explicitly reported no latch inferred for
  ALU, immediate, decoder-control, and next-PC outputs.
- No Yosys warning/error entries were observed in the preserved run log.

Cell counts are specific to these RTL files, Yosys version, and hierarchical
flow. The ALU contains operations not yet selected by the v0.1 decoder, so its
standalone module statistics must not be interpreted as extra ISA support or
as a minimal fully cross-optimized implementation.

## Lint warning disposition

Executed separately:

```sh
make lint-core
```

Verilator 5.050 returned status 0 and one visible `UNUSEDSIGNAL` warning at
`rtl/immediate_generator.sv:2`: instruction bits `[19:12,6:0]` are unused in
this module. I/S/B immediate extraction uses other fields, so the warning is
accepted for this scope. Other core modules still use those instruction bits.

The v0.4 closeout rerun on 2026-09-20 had the same result. The historical
v0.2/day13 output remains in `reports/lint/day13-core.log`; no separate
current-lint artifact is required for this closeout.
`-Wno-fatal` permits completion with warnings; no warning class was hidden.
Revisit this disposition if immediate formats or interfaces change.

## What these results do not establish

- No technology-mapped silicon area or SRAM resource result.
- No target frequency, Fmax, slack, measured critical path, or STA.
- No timing-driven optimization, placement, routing, or physical signoff.
- No equivalence proof or gate-level simulation of the emitted netlist.
- No proof that the supported instructions are correct for all inputs.

The 10 ns cocotb clock is not a frequency result. A plausible single-cycle load
path is register file -> ALU -> external data memory -> writeback; measuring it
requires a defined memory implementation/timing model, library, and constraints.
Basic STA remains part of the later v1.0 engineering work.

## v0.2 rerun

Reproduced on 2026-09-16 with the same WSL Ubuntu-24.04 toolchain after the
decoder extension. The command was:

```sh
yosys -Q -T -l reports/synthesis/v0.2-core.log -p 'read_verilog -sv rtl/rv32i_core.sv rtl/pc.sv rtl/decoder.sv rtl/register_file.sv rtl/immediate_generator.sv rtl/alu.sv; synth -top rv32i_core; check -assert; stat; write_verilog -noattr build/synthesis/v0.2-rv32i_core.v'
```

The command exited 0. The generated log and netlist are ignored artifacts at
`reports/synthesis/v0.2-core.log` and
`build/synthesis/v0.2-rv32i_core.v`.

| Scope | Cells |
| --- | ---: |
| alu | 1483 |
| decoder | 151 |
| immediate_generator | 78 |
| pc | 143 |
| register_file | 3167 |
| rv32i_core, direct contents | 355, including five child instances |
| **Full hierarchy, primitive total** | **5372** |

The hierarchy calculation is `1483 + 151 + 78 + 143 + 3167 + (355 - 5) =
5372`. The register-file and PC storage counts remain 1024 enabled flip-flops
and 32 flip-flops, respectively. The full hierarchy has 0 memory objects;
`check -assert` reports 0 problems; and the combinational conversions report no
inferred latch. These are generic structural observations only. No
technology-specific area, Fmax, slack, STA, placement, routing, or gate-level
equivalence claim is made.

## v0.3 rerun

Reproduced on 2026-09-19 with the same WSL Ubuntu-24.04 toolchain after the
branch decoder/core extension. The command was:

`yosys -Q -T -l reports/synthesis/v0.3-core.log -p 'read_verilog -sv rtl/rv32i_core.sv rtl/pc.sv rtl/decoder.sv rtl/register_file.sv rtl/immediate_generator.sv rtl/alu.sv; synth -top rv32i_core; check -assert; stat; write_verilog -noattr build/synthesis/v0.3-rv32i_core.v'`

The command exited 0. The generated log and netlist are ignored artifacts at
`reports/synthesis/v0.3-core.log` and `build/synthesis/v0.3-rv32i_core.v`.

| Scope | Cells |
| --- | ---: |
| alu | 1429 |
| decoder | 169 |
| immediate_generator | 78 |
| pc | 143 |
| register_file | 3167 |
| rv32i_core, direct contents | 475, including five child instances |
| **Full hierarchy, primitive total** | **5456** |

The v0.3 hierarchy has 1024 register-file enabled flip-flops, 32 PC
flip-flops, 0 memory objects, and 2394 MUX cells. The structural check reports
0 problems, and Yosys reports no inferred latch for the combinational decoder,
branch selector, immediate, ALU, or next-PC processes. These are generic
structural observations only. They do not establish technology-specific area,
Fmax, slack, STA, placement, routing, or gate-level equivalence.

## v0.4 rerun

Reproduced on 2026-09-20 with the same WSL Ubuntu-24.04 toolchain after the
byte/halfword decoder and core extension. The command was:

`yosys -Q -T -l reports/synthesis/v0.4-core.log -p 'read_verilog -sv rtl/rv32i_core.sv rtl/pc.sv rtl/decoder.sv rtl/register_file.sv rtl/immediate_generator.sv rtl/alu.sv; synth -top rv32i_core; check -assert; stat; write_verilog -noattr build/synthesis/v0.4-rv32i_core.v'`

The command exited 0. The generated log and netlist are ignored artifacts at
`reports/synthesis/v0.4-core.log` and
`build/synthesis/v0.4-rv32i_core.v`.

| Scope | Cells |
| --- | ---: |
| alu | 1429 |
| decoder | 172 |
| immediate_generator | 78 |
| pc | 143 |
| register_file | 3167 |
| rv32i_core, direct contents | 1158, including five child instances |
| **Full hierarchy, primitive total** | **6142** |

The hierarchy calculation is `1429 + 172 + 78 + 143 + 3167 + (1158 - 5) =
6142`. The full hierarchy has 1024 `$_DFFE_PP_` register-file flip-flops,
32 PC flip-flops, 2402 MUX cells, 0 memory objects, and 0 procedural processes
after mapping. `check -assert` reported `Found and reported 0 problems.`
Yosys reported no inferred latch for the decoder, immediate, ALU, next-PC,
branch selector, load-data, byte/halfword selection, or write-data/strobe
combinational processes. These are generic structural observations only. No
technology-specific area, Fmax, slack, STA, placement, routing, or gate-level
equivalence claim is made.
