# Synthesis and Timing

## Tools and inputs

Recorded analysis: 2026-09-24 on WSL Ubuntu 24.04.

| Input | Version or identity |
| --- | --- |
| RTL tops | `rv32i_core`, `rv32i_pipeline_core` |
| Lint | Verilator 5.050 |
| Synthesis | Yosys 0.33, revision `2584903a060` |
| STA | OpenSTA 2.6.0 |
| Library | Nangate45 typical, pinned OpenROAD-flow-scripts revision below |

The instruction and data memories are external to both synthesized cores.
No SRAM macro, placement, routing, or extracted wire parasitics are included.
EDA tools and the Nangate45 library are obtained from their upstream
distributions and retain their respective licenses. They are not bundled
with the project source.

## Lint and generic synthesis

```sh
make lint-core lint-pipeline
```

Both lint targets exited successfully. The reviewed `UNUSEDSIGNAL` warning
is for `instr[6:0]` in `immediate_generator.sv`: the opcode is decoded
elsewhere and is not used to assemble immediate fields. `-Wno-fatal` keeps
the warning visible while allowing completion.

From the repository root:

```sh
mkdir -p build/synthesis
core_sources="rtl/rv32i_core.sv rtl/pc.sv rtl/decoder.sv rtl/register_file.sv rtl/immediate_generator.sv rtl/alu.sv"
pipeline_sources="rtl/rv32i_pipeline_core.sv rtl/pipeline_frontend.sv rtl/pipeline_id.sv rtl/pipeline_id_ex.sv rtl/pipeline_ex.sv rtl/pipeline_ex_mem.sv rtl/pipeline_hazard.sv rtl/pipeline_mem.sv rtl/pipeline_mem_wb.sv rtl/decoder.sv rtl/immediate_generator.sv rtl/register_file.sv rtl/alu.sv"
yosys -Q -T -l build/synthesis/core_generic.log -p "read_verilog -sv $core_sources; synth -top rv32i_core; check -assert; stat; write_verilog -noattr build/synthesis/rv32i_core.v"
yosys -Q -T -l build/synthesis/pipeline_generic.log -p "read_verilog -sv $pipeline_sources; synth -top rv32i_pipeline_core -flatten; check -assert; stat; write_verilog -noattr build/synthesis/rv32i_pipeline_core.v"
```

| Core | Generic flow | Cells | Structural-check problems |
| --- | --- | ---: | ---: |
| Single-cycle | Hierarchical | 6,642 | 0 |
| Pipeline | Flattened | 8,626 | 0 |

The generic flows differ in hierarchy handling, so their counts are
structural observations rather than a controlled area comparison. Both lower
the register-file array to logic and flip-flops, with no remaining memory objects.

## Technology mapping

Use this exact
[Nangate45 typical Liberty file](https://raw.githubusercontent.com/The-OpenROAD-Project/OpenROAD-flow-scripts/3a964e13f11a4e435aac01ffa14db0a7d2853720/flow/platforms/nangate45/lib/NangateOpenCellLibrary_typical.lib).

- OpenROAD-flow-scripts revision: `3a964e13f11a4e435aac01ffa14db0a7d2853720`
- SHA-256: `8d540a4d4cf6d09d27c87ad067857a9c0c2eeb023ab7a56e058cd3113db4e9b1`
- Local input: `build/timing/NangateOpenCellLibrary_typical.lib`

After saving the library, run the following in the same shell as the source
lists above:

```sh
mkdir -p build/timing
liberty=build/timing/NangateOpenCellLibrary_typical.lib
sha256sum "$liberty"
yosys -Q -T -q -l build/timing/yosys_map.log -p "read_liberty -lib -ignore_miss_func $liberty; read_verilog -sv $core_sources; hierarchy -check -top rv32i_core; synth -top rv32i_core -flatten; dfflibmap -liberty $liberty; abc -liberty $liberty; clean; check -assert; stat -liberty $liberty; write_verilog -noattr -noexpr -nodec build/timing/rv32i_core_nangate45.v"
yosys -Q -T -q -l build/timing/pipeline_yosys_map.log -p "read_liberty -lib -ignore_miss_func $liberty; read_verilog -sv $pipeline_sources; hierarchy -check -top rv32i_pipeline_core; synth -top rv32i_pipeline_core -flatten; dfflibmap -liberty $liberty; abc -liberty $liberty; clean; check -assert; stat -liberty $liberty; write_verilog -noattr -noexpr -nodec build/timing/rv32i_pipeline_core_nangate45.v"
```

| Metric | Single-cycle | Pipeline |
| --- | ---: | ---: |
| Mapped library cells | 6,267 | 9,556 |
| Liberty cell-area sum | 12,737.410 | 17,515.302 |
| `check -assert` problems | 0 | 0 |

Both mappings use the same flattened flow and library. The pipeline includes
32-bit cycle/retirement counters and retirement-PC observation logic, so the
totals do not represent identically instrumented cores. Liberty cell-area sums
exclude floorplan whitespace, routing, clock trees, memories, and other
physical overhead.

## STA constraints and reproduction

Both SDC files use the following assumptions:

| Constraint | Value |
| --- | --- |
| Clock period | 10.00 ns |
| Setup uncertainty | 0.10 ns |
| External input/output delay, max | 0.50 ns |
| External input/output delay, min | 0.00 ns |
| Input transition | 0.05 ns |
| Output load | 5 fF |
| Clock network | Ideal |
| Library corner | Nangate45 typical |

The pipeline SDC additionally constrains its retirement and counter output
ports. The input/output budgets are illustrative pin constraints; they are
not measured instruction- or data-memory access times.

The recorded OpenSTA executable came from the
[OpenROAD 2024-12-14 Ubuntu 22.04 package](https://github.com/Precision-Innovations/OpenROAD/releases/tag/2024-12-14).
For an OpenSTA installation available as `sta`:

```sh
sta -no_init -no_splash -exit timing/run_sta.tcl > build/timing/single_cycle_sta_10ns.txt 2>&1
sta -no_init -no_splash -exit timing/run_pipeline_sta.tcl > build/timing/pipeline_sta_10ns.txt 2>&1
```

## Complete recorded results

Both complete reports below are from 2026-09-24:

| Metric | Single-cycle | Pipeline |
| --- | ---: | ---: |
| Worst setup slack at 10 ns | +5.202 ns | +6.181 ns |
| Setup WNS / TNS | 0 / 0 ns | 0 / 0 ns |
| Max-slew violations | 49 | 1,576 |
| Max-capacitance violations | 91 | 84 |

Saved STA analysis output:

- [Single-cycle STA](../reports/timing/core_setup_nangate45_typ_10ns.txt)
- [Pipeline STA](../reports/timing/pipeline_setup_nangate45_typ_10ns.txt)

The single-cycle worst reported path is `instr[21]` to
`data_write_data[10]`, with 4.198 ns arrival and 9.400 ns required time.
The pipeline worst path is `_17783_/Q` to `_18710_/D`, with 3.676 ns
arrival and 9.857 ns required time. The latter endpoint maps to bit 0 of the
ID/EX rs1 operand register in this netlist.

The mapped designs have max-slew and max-capacitance violations requiring
electrical-load/transition repair. Per-pin values are in the linked reports.
Positive setup slack alone does not establish timing closure.

## Interpretation

The reported slack applies only to the specified core pins and ideal clock
model. Unresolved electrical limits, absent memory timing, and absent routed
parasitics prevent a defensible whole-CPU Fmax or execution-time speedup claim.
The results also do not establish hold closure, multi-corner timing, clock-tree
quality, physical area, gate-level equivalence, or signoff.

The reproducible performance measurements are the cycle and CPI results in
[verification.md](verification.md). A frequency comparison would require
electrical-rule repair, consistent memory/interface assumptions, and a
rechecked implementation flow for both cores.
