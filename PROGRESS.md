# P1 Development Progress

Last updated: 2026-09-15

## Current Status

**DAY14 / P1 v0.1 — technical and documentation checkpoint closed.**

The 12-instruction single-cycle subset executes PC-indexed programs using
external Python instruction/data memories. The latest full regression passed
19/19 cases across seven groups. Lint warnings were reviewed and generic Yosys
synthesis passed. v0.2-v0.5 are plans only; no corresponding RTL was added.

The owner authorized a combined code/documentation commit and push for this
closeout. No Git tag or GitHub Release was requested. Use `git log`, remote
verification, and the final closeout message for the resulting commit; this
document does not equate a version label with a published release tag.

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

## Current integration boundary and open work

- Core instruction and data ports connect to external Python memory models.
- PC-indexed program execution is implemented; there is no RTL memory macro.
- Default regression retains initial 0, not both initial 0 and initial 5.
- Initial 1 is intentionally unverified. The full runner error-path matrix,
  exhaustive ISA/invalid-encoding coverage, and whole-core reference interpreter
  are not implemented/verified.
- No byte/halfword addressing contract, jump/U/J support, bus handshake, trap,
  pipeline, STA, physical area, or Fmax claim.
- Reports, netlists, and waves are ignored by Git; tracked documents preserve
  the commands and observed summaries. Future runs regenerate local evidence.

## Next-session handoff — start v0.2, not DAY11

DAY: DAY14 / v0.1 closeout.

Completed: 12-instruction core, program tests, one-command regression with logs
and seed, reviewed lint, generic synthesis, and checkpoint documentation.

Owner work demonstrated: RTL and primary test development; PC-indexed
instruction driving; memory-model corrections; regression runner construction.
Concepts discussed but not fully orally assessed are identified in DAY13 above.

Actual verification: 19/19 cases, seven groups, exit 0; reviewed lint warning;
Yosys structural check 0 problems. No initial-1, STA, or Fmax evidence.

Actual bug fixed: DAY11 `LogicArray` dictionary key and correct memory-model
driving/sampling; earlier integration bugs remain recorded under DAY10.

Next start: v0.2-A XOR/XORI/SLTU/SLTIU. ALU operations already exist; inspect
`rtl/decoder.sv` and let the owner add decoding and main test expectations.
Then add shift instructions, followed by v0.3 branches and v0.4 subword memory.
The next session, not this closeout, begins implementation.

Read first: `PROJECT_PLAN.md`, `docs/specification.md`, this handoff, and relevant
RTL/tests. Preserve mentor mode; do not repeat completed exercises or turn
auxiliary Python plumbing into the main learning task. Document edits and Git
operations still follow the owner's explicit scope and authorization.
