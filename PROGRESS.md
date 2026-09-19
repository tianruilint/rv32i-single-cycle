# P1 Development Progress

Last updated: 2026-09-19

## Current Status

**DAY16 / P1 v0.3 — branch implementation checkpoint and
documentation/Git closeout.**

The 27-instruction single-cycle subset executes PC-indexed programs using
external Python instruction/data memories. The latest full regression passed
24/24 cases across seven groups, and the core target passed 11/11. The v0.3
branch implementation is present and its current tests pass, but relational
branch coverage is not complete: the BLT/BGE and BLTU/BGEU reverse directions
and equality boundaries remain unverified.

The owner authorized a combined code/documentation commit and push for this
v0.3 closeout. No Git tag or GitHub Release was requested. Use `git log`,
remote verification, and the final closeout message for the resulting commit;
this document does not equate a version label with a published release tag.

The authoritative working repository is `D:\projects\rv32i-single-cycle`
(`/mnt/d/projects/rv32i-single-cycle` in WSL).

## DAY03 — Register File

### Completed deliverables

- `rtl/register_file.sv`
- `tb/test_register_file.py`
- `Makefile` target: `test-register-file`

### Implemented behavior

- 32 registers, each 32 bits wide.
- Two combinational read ports: `rs1` and `rs2`.
- One synchronous write port using `always_ff @(posedge clk)`.
- Writes require `reg_write = 1`.
- Reads from `x0` always return zero.
- Writes to `x0` are ignored.
- No reset is implemented; tests do not assume an initial value for `x1`-`x31`.

### Verification evidence

Command executed on 2026-09-12:

```bash
make test-register-file
```

Observed environment and result:

- Python 3.12.4
- cocotb 2.0.1
- Verilator 5.050
- Tests: 5
- Pass: 5
- Fail: 0
- Skip: 0
- Simulation time: 69 ns

Verified cases:

1. Normal write to and read from `x5`.
2. Reading `x0` returns zero.
3. A write to `x0` is ignored.
4. The two read ports return different known registers simultaneously.
5. `reg_write = 0` preserves the previous register value across a clock edge.

RTL lint command executed on 2026-09-12:

```bash
verilator --lint-only --Wall -Wno-fatal --top-module register_file rtl/register_file.sv
```

Lint exited successfully. That recorded run reported an `EOFNEWLINE` warning;
post-DAY03 inspection confirms that `rtl/register_file.sv` now ends with a
newline, so the warning is no longer part of the current baseline.

Fault injection was completed by the owner. The injected-failure log was not
retained in this progress record, and a full cross-module regression was not
required for this DAY03 closeout by current instruction.

### Concepts understood and practiced

- `wire`, `reg`, and `logic` describe signal/net or variable categories; they do
  not by themselves determine whether hardware is combinational or sequential.
- Register File reads are combinational, while writes occur on the rising clock edge.
- `rs1` and `rs2` are read addresses; `rd` is the write destination address.
- `rd_data` is an input to the Register File; `rs1_data` and `rs2_data` are outputs.
- cocotb `assert` compares an observed DUT result against an expected result.
- `Clock`, `RisingEdge`, and `Timer` coordinate Python stimulus with simulation time.
- The Makefile is understood as a reproducible build/test entry point rather than
  a primary RTL design topic.

### Post-DAY03 baseline

- Commit `0ad6a81` (`day03: implement and verify register file`) is present on
  both local `main` and `origin/main`.
- `tb/test_register_file.py` contains one definition of
  `test_write_to_x0_is_ignored`.
- `rtl/register_file.sv` ends with a newline.
- The repository contains verified component RTL through DAY03, but no
  integrated processor or instruction-execution claim.

## DAY04 — Program Counter

### Completed deliverables

- `rtl/pc.sv`
- `tb/test_pc.py`
- `Makefile` target: `test-pc`

### Implemented behavior

- 32-bit `current_pc` output.
- Synchronous, active-high reset to zero.
- Reset has priority over target selection.
- Sequential update advances the PC by four.
- `take_target = 1` loads `target_pc` on the next rising edge.
- 32-bit addition naturally wraps around.

At DAY04, the instruction-memory portion was deferred. DAY11 later supplied
a PC-indexed Python instruction-memory model; no RTL instruction ROM is claimed.

### Verification evidence

Command executed on 2026-09-13:

```bash
make test-pc
```

Observed result with Verilator 5.050 and cocotb 2.0.1:

- Tests: 3
- Pass: 3
- Fail: 0
- Simulation time: 83 ns

Verified reset, sequential `+4`, target loading, reset priority, and 32-bit
wraparound. A PC-only Verilator lint run exited with code 0 and no warning.

## DAY05 — Immediate Generator

### Completed deliverables

- `rtl/immediate_generator.sv`
- `tb/test_immediate_generator.py`
- `Makefile` target: `test-immediate-generator`

### Implemented behavior

- Internal format codes: I = `2'b00`, S = `2'b01`, B = `2'b10`.
- Correct I-type sign extension.
- Correct S-type field reassembly and sign extension.
- Correct B-type field reassembly and sign extension, with bit 0 fixed to zero.
- Unsupported format codes produce zero.

U-type and J-type immediates are deferred.

### Verification evidence

Command executed on 2026-09-13:

```bash
make test-immediate-generator
```

Observed result:

- cocotb test cases: 1
- Pass: 1
- Fail: 0
- Tested vectors: 6
- Simulation time: 6 ns

The immediate-generator lint run exited with code 0. Verilator reported a
nonfatal `UNUSEDSIGNAL` warning for instruction bits `[19:12]` and `[6:0]`,
which are not consumed by the currently implemented I/S/B formats.

## DAY06 — Decoder

### Completed deliverables

- `rtl/decoder.sv`
- `tb/test_decoder.py`
- `Makefile` target: `test-decoder`

### Implemented behavior

- Inputs: `opcode`, `funct3`, and `funct7`.
- Outputs: register write, ALU-source select, memory write, result-source
  select, branch, immediate type, and ALU operation.
- R-type support: ADD, SUB, AND, OR, SLT.
- I-type support: ADDI, ANDI, ORI, SLTI.
- Load/store/branch support: LW, SW, BEQ.
- R-type decoding checks both `funct3` and `funct7` where required.
- Unsupported or invalid encodings retain safe inactive defaults.

### Verification evidence

Command executed on 2026-09-13:

```bash
make test-decoder
```

Observed result:

- cocotb test cases: 1
- Pass: 1
- Fail: 0
- Tested vectors: 9
- Simulation time: 9 ns

The decoder-only lint run exited with code 0 and no warning. The current nine
vectors do not directly exercise valid OR or SLT decoding, and no decoder
random test or fault-injection cycle was required for this closeout.

## DAY07 — Datapath Integration Part 1

### Completed deliverables

- `rtl/rv32i_core.sv`
- `tb/test_core.py`
- `Makefile` targets: `lint-core`, `test-core`, and `waves-core`

### Implemented behavior

- Connected the PC, decoder, register file, immediate generator, and ALU.
- Extracted opcode, funct3, funct7, rs1, rs2, and rd from the instruction.
- Added the ALU operand-B selection and ALU-result writeback path.
- Executed and checked ADD, ADDI, SUB, AND, ANDI, OR, ORI, SLT, and SLTI.
- Prevented register-file writes while reset is asserted.

## DAY08 — Load / Store

### Completed deliverables

- Core data-memory interface: `data_read_data`, `data_addr`,
  `data_write_data`, and `data_write_en`.
- `tb/test_lw_sw.py`.

### Implemented behavior

- LW computes `rs1 + immediate`, receives external read data, and selects it
  for register writeback.
- SW computes `rs1 + immediate` and exposes the address, rs2 write data, and
  write-enable signal.
- The directed test uses a Python dictionary as the external data-memory model;
  no RTL data-memory module is claimed.
- The verified sequence stored 42 at byte address 72 and loaded it into x3.

## DAY09 — Branch

### Completed deliverables

- BEQ comparison, branch decision, target-address calculation, and PC control.
- `tb/test_beq.py`.

### Implemented behavior

- `branch_taken` requires both a decoded BEQ and equal rs1/rs2 values.
- The taken target is the current branch PC plus the sign-extended B immediate.
- Tests cover taken, not-taken, and negative-offset branches.
- Tests check that BEQ does not enable register-file or data-memory writes.

## DAY10 — P1 v0.1 Core Integration

### Supported instruction subset

- ADD, ADDI, SUB
- AND, ANDI, OR, ORI
- SLT, SLTI
- LW, SW
- BEQ

### Verification evidence

Commands executed on 2026-09-14:

```bash
make lint-core
make test-core
```

Observed core regression result:

- cocotb test cases: 4
- Pass: 4
- Fail: 0
- Skip: 0
- Simulation time: 284 ns

The four test cases are the arithmetic chain, reset write blocking, LW/SW, and
BEQ. Together they exercise the 12 supported instruction types; this is not a
claim of 12 separate tests.

`make lint-core` exited successfully. The remaining nonfatal warning reports
instruction bits unused by the currently implemented immediate formats.

`make waves-core` was executed on 2026-09-13 with the same four passing test
cases and generated `waves/core.fst`.

### Actual development bugs resolved

- Register write enable was initially independent of decoder control.
- The initial ALU/writeback assignment direction was reversed.
- The first LW/SW test treated a Python memory dictionary as a DUT hierarchy
  object and supplied load data at the wrong time.
- The first BEQ attempt used procedural `if` statements at module scope and
  duplicated PC-update responsibility outside `pc.sv`.

### Git baseline

- Commit `16f4a50` (`day07-10: integrate and verify rv32i core`) is present on
  local `main` and `origin/main`.

## DAY11 — PC-indexed Program Execution

Owner-written deliverable: `tb/test_program.py`, included in the default
`CORE_TEST_MODULES` by the Makefile.

- `test_program`: PC-indexed ADDI/ADDI/ADD program; `x3=12`, `PC=12`.
- `test_program2`: loop, SW to byte address 64, LW into x3, end sentinel PC=32.
  The saved version initializes the counter to 0 and verifies
  `x1=x2=x3=memory[64]=0`. A 40-iteration bound plus final PC assertion detects
  failure to reach the expected endpoint.
- The initial-5 version previously passed with `x1=0`, `x2=x3=memory[64]=15`,
  `PC=32`. The retained six-case report is
  `reports/day11-program2-review2/core.xml` (586 ns total).
  Initial 5 is historical evidence, not a separate current regression case.
- Initial 1 was explicitly skipped by the owner. Do not silently add or claim it.

Actual bug: a DUT `LogicArray` was used as a dictionary key, producing
`TypeError: unhashable type: 'LogicArray'` in `test_program2` at 241 ns.
The failure is retained in `reports/day11-program2-review1/core.xml`.
The owner converted the address to `int`, supplied load data to the DUT input,
and kept memory sampling/updates aligned with settling and the committing edge.
See `docs/bug_diary.md` for the evidence boundary.

The initial-zero waveform run passed 1/1 selected case (51 ns):
`reports/day11-zero-wave/core-waves.xml` and
`waves/day11-zero-wave/core.fst` (1931 bytes). The inspected PC sequence was
`0 -> 4 -> 8 -> 24 -> 28 -> 32`; SW/LW used address 64 and data 0.
This is a historical trace, not a new waveform run during DAY14.

## DAY12 — One-command Regression

The owner rebuilt `scripts/run_regression.py` incrementally in mentor mode
after rejecting an earlier complete assistant implementation.

Implemented: invoke seven Make targets, inspect return codes, read XML cases,
count failure/error/skipped elements, track failed targets, print totals,
return a process status, save per-target stdout/stderr logs, and forward a
chosen `COCOTB_RANDOM_SEED`.

Latest closeout command executed on 2026-09-15:

```sh
make regression SEED=20260915
```

| Group | Cases | Pass | Fail | Skip |
| --- | ---: | ---: | ---: | ---: |
| Full adder | 1 | 1 | 0 | 0 |
| ALU | 2 | 2 | 0 | 0 |
| Register file | 5 | 5 | 0 | 0 |
| PC | 3 | 3 | 0 | 0 |
| Immediate generator | 1 | 1 | 0 | 0 |
| Decoder | 1 | 1 | 0 | 0 |
| Core | 6 | 6 | 0 | 0 |
| **Total** | **19** | **19** | **0** | **0** |

Exit status: 0. Core simulated time: 386 ns. Seven logs under
`reports/regression/` confirm supplied cocotb seed 20260915. The ALU's separate
`random.Random(20260906)` remains independently fixed. Nineteen cases do not
mean nineteen instructions, nineteen vectors, or exhaustive coverage.

`read_report()` was also run on the real historical failure report and returned
`(2, 1, 1)` for total/failed/skipped. Runner make-failure, missing/malformed XML,
empty-report, and skipped-case exit paths are reviewed but not all dynamically
validated. On make failure the runner skips XML parsing; totals describe reports
actually parsed, while `Failed targets` determines overall failure.

Deferred: assembly-to-image automation, per-run manifest/history, and automatic
failure-wave generation. These are not prerequisites for resuming owner-written
instruction extensions. Current logs overwrite the previous same-target logs.

## DAY13 — Lint and Generic Synthesis

Executed with Verilator 5.050 and Yosys 0.33; rerun and preserved at DAY14.

- `make lint-core`: exit 0, one `UNUSEDSIGNAL` warning for the immediate
  generator's `instr[19:12,6:0]`. Reviewed against I/S/B bit extraction; accepted
  without disabling the warning. No RTL change was needed.
- Yosys `synth -top rv32i_core; check -assert; stat`: exit 0, structural check
  reports 0 problems; no latch inferred from the combinational processes.
- Hierarchical total: 5314 generic cells; register file: 1024 enabled flip-flops
  and 1984 MUX cells; PC: 32 flip-flops. Memory objects after mapping: 0.
- Saved logs: `reports/lint/day13-core.log`,
  `reports/synthesis/day13-core.log`.
- Saved generated netlist: `build/synthesis/rv32i_core.v`.
- Reproduction, hierarchy, and limitations: `docs/synthesis.md`.

These are generic structural results, not technology-mapped area, SRAM count,
Fmax, timing closure, or functional equivalence. No STA or gate-level regression
has been performed.

Learning evidence: the owner identified flip-flops as state and the 32x32 array
as 1024 bits. Address-controlled read selection, the MUX count, and the
distinction between simulation/lint/synthesis were explained. A complete
independent oral explanation has not been assessed; do not claim mastery solely
from tool success.

## DAY14 — v0.1 Documentation and Git Closeout

Owner instruction: stop at v0.1, comprehensively update repository documents,
then commit and push the existing DAY11–12 work and documentation together.
The earlier word "pull" was corrected by the owner to "push".

Before edits, an explicitly requested fast-forward-only pull used the existing
WSL GitHub CLI login and reported `Already up to date.` The pre-closeout local
HEAD was `6118046`; `origin/main` was `16f4a50`. Local changes were preserved.

Updated the README, implemented specification, project plan, progress record,
environment notes, and real bug diary. Added datapath, control table,
verification scope, synthesis reproduction, and concise learning-boundary
documents. Saved tool evidence under ignored generated-output directories.
No v0.2 instruction support, RTL redesign, or test-logic replacement was made.

Git commit/push completion is verified separately after all document edits.
No tag or GitHub Release is part of this authorization.

## DAY15 — P1 v0.2 Logic, Unsigned Comparison, and Shift Instructions

Closed on 2026-09-16. This checkpoint extends the owner-written single-cycle
decoder and core tests without starting branches, subword memory, jumps, or the
pipeline.

### Owner-written deliverables

- `rtl/decoder.sv`: CPU-level decode for XOR/XORI, SLTU/SLTIU, SLL/SLLI,
  SRL/SRLI, and SRA/SRAI.
- `tb/test_decoder.py`: legal R/I encodings plus invalid immediate-shift
  `funct7` boundaries. This is one cocotb case containing 22 vectors.
- `tb/test_core.py`: integrated XOR/unsigned-comparison and shift execution
  tests, including shift amount 31 and a register shift source of 32.

### Verification evidence

Full regression executed on 2026-09-16:

```sh
make regression SEED=20260916
```

| Group | Cases | Pass | Fail | Skip |
| --- | ---: | ---: | ---: | ---: |
| Full adder | 1 | 1 | 0 | 0 |
| ALU | 2 | 2 | 0 | 0 |
| Register file | 5 | 5 | 0 | 0 |
| PC | 3 | 3 | 0 | 0 |
| Immediate generator | 1 | 1 | 0 | 0 |
| Decoder | 1 | 1 | 0 | 0 |
| Core | 8 | 8 | 0 | 0 |
| **Total** | **21** | **21** | **0** | **0** |

Exit status: 0. `make lint-core` also exited 0 with the previously reviewed
nonfatal immediate-generator `UNUSEDSIGNAL` warning. The v0.2 Yosys rerun
reported 0 structural problems from `check -assert`, no inferred latch, and
5372 generic cells in the full hierarchy. These are generic structural counts,
not technology-specific area, Fmax, or STA evidence.

### Bug found and fixed

An invalid immediate-shift encoding was initially accepted because the decoder
treated the shift-immediate `funct7`-like field as don't-care. The owner added
the required `0000000` qualification for SLLI/SRLI and `0100000` qualification
for SRAI, then added illegal-encoding vectors. Ordinary I-type arithmetic still
treats the upper immediate bits as data. The targeted decoder failure and the
fix are recorded in `docs/bug_diary.md`.

### Knowledge and boundary

The owner implemented the core decoder/test changes and reviewed the distinction
between register write enable and the external memory `data_write_en`, signed
versus unsigned comparison, logical versus arithmetic right shift, and RV32's
low-five-bit register shift amount rule. Initial-1 loop behavior remains
explicitly unverified. There is still no byte/halfword memory, remaining branch
group, U/J instruction, whole-core reference interpreter, formal proof, STA,
physical area, or Fmax result.

## DAY16 — P1 v0.3 Branch Extension

Owner-written v0.3 changes are present in `rtl/decoder.sv`,
`rtl/rv32i_core.sv`, `tb/test_decoder.py`, and `tb/test_beq.py`. The decoder
adds `branch_type` codes `NONE=0`, `BEQ=1`, `BNE=2`, `BLT=3`, `BGE=4`,
`BLTU=5`, and `BGEU=6`. The core selects equality, inequality, signed
less-than, unsigned less-than, or the corresponding inverse and keeps branch
register/memory writes inactive.

The primary branch tests cover BNE equal/not-equal cases, signed `-1` versus
`1` with BLT taken and BGE not-taken, and unsigned `0xffffffff` versus `1`
with BLTU not-taken and BGEU taken. Each new branch case checks `rf_we=0`
and `data_write_en=0`.

Observed verification on 2026-09-19:

- `make lint-core`: exit 0; one visible, accepted `UNUSEDSIGNAL` warning for
  the I/S/B immediate generator's unused `instr[19:12,6:0]`.
- `make test-core`: 11/11 cases passed, 0 failed, 0 skipped.
- `make regression SEED=20260919`: seven groups, 24/24 cases passed, 0
  failed, 0 skipped, exit status 0.
- `tb/test_decoder.py`: one cocotb case with 29 decoder vectors.
- The implemented subset is 27 instruction types, not 27 test cases.
- The v0.3 synthesis reproduction passed with 5456 generic cells,
  `check -assert` reporting 0 problems, and no inferred combinational latch.
  This is generic structural evidence only; there is no STA, Fmax, physical
  area, or gate-level equivalence result.

Known v0.3 acceptance gap: BLT/BGE have not been tested with the opposite
ordering that makes BGE taken and BLT not-taken; BLTU/BGEU have not been
tested with the opposite ordering that makes BLTU taken and BGEU not-taken.
Equality boundaries for all four relational branches are also unverified.
The current passing regression must not be described as exhaustive branch
acceptance.

Intermediate review findings recorded for continuity were the `NOEN` spelling
typo for the `NONE` branch code, a duplicated BEQ expected vector during
decoder-vector editing, a trailing-comma/`NULLPORT` interface issue while
adding the new port, and the need to complete the `branch_type` connection
through decoder and core. These were corrected before the current lint,
regression, and synthesis runs; no such current source error is claimed.

The owner wrote the v0.3 core RTL and primary branch tests. This closeout
records observed behavior and evidence boundaries; passing tools do not by
themselves establish a complete independent explanation of every branch
corner case.

## Current integration boundary and open work

- Core instruction and data ports connect to external Python memory models.
- PC-indexed program execution is implemented; there is no RTL memory macro.
- Default regression retains initial 0, not both initial 0 and initial 5.
- Initial 1 is intentionally unverified. The full runner error-path matrix,
  exhaustive ISA/invalid-encoding coverage, and whole-core reference interpreter
  are not implemented/verified.
- No byte/halfword addressing contract, jump/U/J support, bus handshake, trap,
  pipeline, STA, physical area, or Fmax claim.
- The planned 2026-09-18 v2.0 target date has passed, but this repository is
  still at v0.3. The schedule does not lower the verification standard or
  turn this checkpoint into a v2.0 release.
- Reports, netlists, and waves are ignored by Git; tracked documents preserve
  the commands and observed summaries. Future runs regenerate local evidence.

## Next-session handoff — start v0.4, not v0.3

DAY: DAY16 / v0.3 implementation checkpoint and documentation/Git closeout.

Completed: 27-instruction single-cycle core, v0.3 branch selection with
signed/unsigned comparisons, owner-written directed branch cases, 24/24
full-regression evidence, current lint, v0.3 generic synthesis, and updated
checkpoint documents.

Owner work demonstrated: branch decoder extension, `branch_type` interface
completion, signed versus unsigned comparison selection, no-side-effect
expectations, PC-indexed instruction driving, and external memory-model
behavior. The owner has not claimed an independent whole-core reference model,
formal proof, exhaustive branch coverage, STA, or physical PPA result.

Still explicitly unverified: initial-1 loop behavior; the reverse and
equality boundaries for BLT/BGE/BLTU/BGEU; exhaustive ISA/invalid-encoding
coverage; and the full regression-runner error matrix. Initial 5 remains
historical evidence only, while the current default loop uses initial 0.

v0.4 remains a plan only. First define the byte-addressed little-endian
contract, including low-address-bit lane selection, write strobes that preserve
unwritten bytes, signed/zero extension, and the alignment/access-exception
behavior. Only after that contract is agreed should the owner change
`rtl/rv32i_core.sv` and the Python data-memory model and add the corresponding
tests. The first file to revisit is `docs/specification.md`; the first RTL
module after that contract is `rtl/rv32i_core.sv`.

Read first: `PROJECT_PLAN.md`, `README.md`, `docs/specification.md`, this
handoff, and the current RTL/tests. Preserve mentor mode; do not repeat
completed exercises or turn the v0.4 plan into implementation during the next
handoff.
