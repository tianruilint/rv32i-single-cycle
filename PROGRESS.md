# P1 Development Progress

Last updated: 2026-09-14

## Current Status

**DAY10 — P1 v0.1 Core Integration: COMPLETE**

The instruction-level single-cycle core supports the planned 12-instruction
subset and has passed its integrated lint, regression, and waveform runs.
Program-level execution and the formal P1 v0.1 release remain pending.

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

The instruction-memory portion originally associated with instruction fetch is
deferred and has not been implemented.

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

## Current integration boundary

- The integrated core accepts an external `instr` input; instruction memory is
  not implemented yet.
- Data memory is represented by an external cocotb model, not an RTL module.
- Instruction-level sequences are automated, but PC-indexed program execution
  has not been completed.
- P1 v0.1 has not reached its formal DAY14 release.

## Next Starting Point

**DAY11 — Program Execution**

Begin DAY11 by defining a PC-indexed instruction-memory model and one small
program whose final register and memory state can be checked automatically.
