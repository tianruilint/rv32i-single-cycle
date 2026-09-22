# v1.0 preliminary core-only setup analysis (2026-09-22)

This is an educational, **pre-layout** STA run of the single-cycle core, not
timing signoff or a measured CPU/SoC Fmax. The instruction and data memories
remain external to the RTL netlist.

## Inputs and reproduction

- RTL: `rtl/rv32i_core.sv`, `rtl/pc.sv`, `rtl/decoder.sv`,
  `rtl/register_file.sv`, `rtl/immediate_generator.sv`, `rtl/alu.sv` at
  `006f837` (no RTL edits for this analysis).
- Yosys 0.33 (`2584903a060`) for mapping; standalone OpenSTA 2.6.0 from the
  [OpenROAD 2024-12-14 Ubuntu 22.04 release](https://github.com/Precision-Innovations/OpenROAD/releases/tag/2024-12-14)
  for timing analysis. Both run under WSL Ubuntu 24.04; OpenSTA was extracted
  project-locally rather than installed system-wide.
- [Nangate45 typical Liberty](https://raw.githubusercontent.com/The-OpenROAD-Project/OpenROAD-flow-scripts/3a964e13f11a4e435aac01ffa14db0a7d2853720/flow/platforms/nangate45/lib/NangateOpenCellLibrary_typical.lib),
  ORFS commit `3a964e13f11a4e435aac01ffa14db0a7d2853720`, SHA-256
  `8d540a4d4cf6d09d27c87ad067857a9c0c2eeb023ab7a56e058cd3113db4e9b1`.
  Copy this file to `build/timing/NangateOpenCellLibrary_typical.lib`.
- The mapped netlist and mapping log are generated, ignored files under
  `build/timing/`. From the repository root in WSL:

```sh
yosys -Q -T -q -l build/timing/yosys_map.log -p 'read_liberty -lib -ignore_miss_func build/timing/NangateOpenCellLibrary_typical.lib; read_verilog -sv rtl/rv32i_core.sv rtl/pc.sv rtl/decoder.sv rtl/register_file.sv rtl/immediate_generator.sv rtl/alu.sv; hierarchy -check -top rv32i_core; synth -top rv32i_core -flatten; dfflibmap -liberty build/timing/NangateOpenCellLibrary_typical.lib; abc -liberty build/timing/NangateOpenCellLibrary_typical.lib; clean; check -assert; stat -liberty build/timing/NangateOpenCellLibrary_typical.lib; write_verilog -noattr -noexpr -nodec build/timing/rv32i_core_nangate45.v'
LD_LIBRARY_PATH="$PWD/build/timing/tclreadline/usr/lib/x86_64-linux-gnu" build/timing/openroad/usr/bin/sta -no_init -no_splash -exit timing/run_sta.tcl
```

The extracted OpenSTA build also needed a local `tcl-tclreadline` library at
the `LD_LIBRARY_PATH` shown above. Its nonfatal `Failed to load
tclreadline.tcl` startup message does not stop the batch STA run. Constraint
assumptions are in `timing/constraints.sdc`: 10 ns clock target, 0.10 ns setup
uncertainty, illustrative 0.50 ns input/output external delays, 0.05 ns input
transition, and 5 fF output load. The clock is ideal; memory access delays,
wire parasitics, placement, routing, and other process corners are absent.

## Observed result and limits

The mapped netlist passed Yosys `check -assert` and contains 6,267 library
cells. OpenSTA `check_setup -verbose` passed. At the assumed 10 ns period,
setup worst slack was **+5.202 ns** (WNS 0.00 ns, TNS 0.00 ns). The slowest
reported path ran from `instr[21]` to `data_write_data[10]`, with 4.198 ns
data arrival against a 9.400 ns required time. Input-to-register worst slack
was +5.264 ns; register-to-register +6.536 ns; `data_read_data`-to-register
+8.841 ns. The full console output is retained in Git at
`reports/timing/core_setup_nangate45_typ_10ns.txt`.

**This is not timing closure.** The mapped netlist has 15 reported
maximum-slew violations, including the worst path's high-fanout driver:
0.845 ns slew against a 0.199 ns library limit. The report contains no
maximum-capacitance or fanout table; do not interpret that as proof of no such
problems. The 10 ns target passing in this incomplete, idealized model does
not establish a real maximum frequency or the period of a CPU with memories.
The Yosys mapping log also warns that scan-flop Liberty expressions were
skipped; the mapped design uses ordinary `DFF_X1` cells. No gate-level
functional equivalence or post-layout analysis was performed.
