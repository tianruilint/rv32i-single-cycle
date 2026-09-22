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

The v1.0 rerun with Verilator 5.050 returned status 0 and one visible
`UNUSEDSIGNAL` warning at `rtl/immediate_generator.sv:2`: opcode bits `[6:0]`
are unused in this module. I/S/B/U/J immediate extraction uses the other
instruction fields, so the warning is accepted for this scope. The core and
decoder still use the opcode bits.

The v0.4 closeout rerun on 2026-09-20 reported `[19:12,6:0]` because U/J
formats were not yet present. The historical
v0.2/day13 output remains in `reports/lint/day13-core.log`; the v1.0 lint
result was observed directly from `make lint-core` on 2026-09-22.
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
This section describes the historical generic run. Basic core-only STA was
subsequently performed for v1.0; see the final section below.

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

## v0.5 rerun

Reproduced on 2026-09-21 with the same WSL Ubuntu-24.04 toolchain after the
U/J immediate, upper-immediate, and jump-path extension. The command was:

`yosys -Q -T -l reports/synthesis/v0.5-core.log -p 'read_verilog -sv rtl/rv32i_core.sv rtl/pc.sv rtl/decoder.sv rtl/register_file.sv rtl/immediate_generator.sv rtl/alu.sv; synth -top rv32i_core; check -assert; stat; write_verilog -noattr build/synthesis/v0.5-rv32i_core.v'`

The command exited 0. The generated log and netlist are ignored artifacts at
`reports/synthesis/v0.5-core.log` and
`build/synthesis/v0.5-rv32i_core.v`.

| Scope | Cells |
| --- | ---: |
| alu | 1429 |
| decoder | 206 |
| immediate_generator | 216 |
| pc | 143 |
| register_file | 3167 |
| rv32i_core, direct contents | 1486, including five child instances |
| **Full hierarchy, primitive total** | **6642** |

The hierarchy calculation is `1429 + 206 + 216 + 143 + 3167 + (1486 - 5) =
6642`. The full hierarchy has 1024 `$_DFFE_PP_` register-file flip-flops,
32 PC flip-flops, 2465 MUX cells, 0 memory objects, and 0 procedural processes
after mapping. `check -assert` reported `Found and reported 0 problems.` Yosys
reported no inferred latch for the decoder, I/S/B/U/J immediate generator,
ALU, next-PC/jump target, branch selector, load-data, byte/halfword selection,
writeback, or write-data/strobe combinational processes.

The increase from the historical v0.4 count of 6142 reflects this generic flow
over changed RTL; it is not a physical area comparison. No technology-specific
area, Fmax, slack, STA, placement, routing, or gate-level equivalence claim is
made.

## v1.0 technology-mapped core and basic STA

On 2026-09-22, the unchanged v0.5 RTL was flattened and mapped with Yosys
0.33 to the pinned Nangate45 typical Liberty library. `check -assert` reported
0 structural problems, and `stat` reported 6267 library cells, including 1056
`DFF_X1` cells. The mapping log is `build/timing/yosys_map.log`, the generated
netlist is `build/timing/rv32i_core_nangate45.v`, and the exact mapping command
and library SHA-256 are in [timing/README.md](../timing/README.md). This cell
count and the Liberty cell-area sum are not a placed or routed silicon area.

Yosys warned that unsupported scan-flop pin expressions were skipped during
mapping; ordinary `DFF_X1` cells were mapped. ABC also reported multi-output
gates in the library and that its network was combinational. Neither warning
was hidden. The mapped design passed the final structural check, but there
was no gate-level functional-equivalence run.

OpenSTA 2.6.0 read that netlist and library with
`timing/constraints.sdc`. `check_setup -verbose` passed. At an illustrative
10 ns period, worst setup slack was +5.202 ns (WNS/TNS 0), from `instr[21]`
to `data_write_data[10]`. The saved raw report is
`reports/timing/core_setup_nangate45_typ_10ns.txt`. Fifteen maximum-slew
violations remain, including 0.845 ns against a 0.199 ns library limit on a
high-fanout driver. External instruction/data memory timing, clock-tree delay,
wire parasitics, physical placement, and other process corners are absent.
This is a basic educational core-only timing analysis, not timing closure,
an achievable whole-CPU Fmax, or physical signoff.
