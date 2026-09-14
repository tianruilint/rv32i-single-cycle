# RV32I Single-Cycle Processor

An owner-written educational 32-bit single-cycle CPU with automated
SystemVerilog/Verilator/cocotb verification.

**Current checkpoint: DAY14 / P1 v0.1, closed on 2026-09-15.**
This is a verified **RV32I-subset** core, not a complete RV32I implementation.
v0.2 and later work has not started. Version names identify development
milestones; no Git tag or GitHub Release is implied.

## Implemented scope

The integrated core supports these **12 instruction types**:

- ADD, ADDI, SUB
- AND, ANDI, OR, ORI
- SLT, SLTI
- LW, SW
- BEQ

`rtl/rv32i_core.sv` connects the PC, decoder, register file, immediate generator,
ALU, load/store interface, writeback selection, and BEQ path. Instruction and
data memories are **external Python models**, not synthesized RAM modules.
The program tests fetch instruction words using the DUT's `current_pc`.

The ALU also implements XOR, shifts, and unsigned comparison at component level.
Their CPU instruction decoding is not implemented in v0.1.
Read the [specification](docs/specification.md),
[datapath diagram](docs/datapath.md), and
[control table](docs/control_table.md) for the precise boundary.

## Quick start

Run commands in WSL Ubuntu from the repository root. The maintained local
checkout is `D:\projects\rv32i-single-cycle`, mounted at
`/mnt/d/projects/rv32i-single-cycle`; the ChatGPT project mirror is separate.

The verified tools are Verilator 5.050, cocotb 2.0.1, pytest 8.3.5, GTKWave
3.3.116, and Yosys 0.33. See [environment notes](docs/environment.md).
For a fresh Python environment only:

```sh
cd /mnt/d/projects/rv32i-single-cycle
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements.txt
```

Do not recreate an existing working environment. Normal checks are:

```sh
make env
make lint-core
make regression SEED=20260915
```

`make regression` runs all seven test groups, reports case counts and failed
targets, and returns a nonzero status on a detected failure. It must run from
the repository root. `make test` alone still tests **only the full adder**.

## Verified v0.1 results

Full regression rerun on 2026-09-15:

| Target | cocotb cases | Passed | Failed | Skipped |
| --- | ---: | ---: | ---: | ---: |
| `test` | 1 | 1 | 0 | 0 |
| `test-alu` | 2 | 2 | 0 | 0 |
| `test-register-file` | 5 | 5 | 0 | 0 |
| `test-pc` | 3 | 3 | 0 | 0 |
| `test-immediate-generator` | 1 | 1 | 0 | 0 |
| `test-decoder` | 1 | 1 | 0 | 0 |
| `test-core` | 6 | 6 | 0 | 0 |
| **Total** | **19** | **19** | **0** | **0** |

The process exited with status 0. These are test-case counts, not instruction
counts or coverage percentages. Core simulation time was 386 ns; this is not
an implementation-performance measurement.

The six core cases check arithmetic, reset write blocking, LW/SW, BEQ,
PC-indexed straight-line execution, and a zero-initialized loop/store/load
program. The straight-line program finishes with `x3=12`, `PC=12`.
The currently saved second program finishes with
`x1=x2=x3=memory[64]=0`, `PC=32`.

The initial-5 loop previously passed with sum 15, but that variant was replaced
by initial 0 in the saved test. It is **historical evidence, not an additional
case in today's regression**. Initial 1 was explicitly skipped by the owner
and remains unverified. See the
[verification plan and evidence](docs/verification_plan.md).

Each group refreshes its XML in `reports/`. The runner saves stdout and stderr
to `reports/regression/<target>.log`, overwriting that target's previous log.
All seven latest logs confirm supplied cocotb seed `20260915`.
The ALU's independent reference-vector generator uses fixed seed `20260906`;
changing `SEED` does not change that generator.

## Targeted tests and waveforms

```sh
make test-alu
make test-register-file
make test-pc
make test-immediate-generator
make test-decoder
make test-core
make waves-core
```

`test-core` and `waves-core` select
`test_core,test_lw_sw,test_beq,test_program`. Wave generation is explicit, not
automatic on failure. `waves-core` writes `waves/core.fst`; inspect it with
GTKWave. Run waveform targets serially: they use the shared `dump.fst` name.

The retained DAY11 initial-zero trace is
`waves/day11-zero-wave/core.fst`. Its PC path was inspected as
`0 -> 4 -> 8 -> 24 -> 28 -> 32`, including SW/LW at byte address 64.

## Lint, synthesis, and timing boundary

`make lint-core` exits successfully with one reviewed `UNUSEDSIGNAL` warning:
the I/S/B immediate generator does not consume `instr[19:12,6:0]`.
No warning class was disabled. `-Wno-fatal` keeps warnings visible while
allowing the command to complete; status 0 does not mean warning-free RTL.

Generic Yosys synthesis and `check -assert` succeeded with 0 reported structural
problems. The recorded hierarchy contains **5314 generic cells**, including
1024 register-file enabled flip-flops and 32 PC flip-flops. No latch was inferred
from the combinational processes. These are tool/run-specific structural
counts, **not silicon area or Fmax**. No target technology library, STA,
post-layout timing, or gate-level equivalence result is claimed.

The [synthesis record](docs/synthesis.md) contains the exact command, hierarchy,
warning disposition, and reproduction instructions. Generated evidence:

- `reports/lint/day13-core.log`
- `reports/synthesis/day13-core.log`
- `build/synthesis/rv32i_core.v` (generated netlist, not hand-written source)

## Repository map

| Path | Purpose |
| --- | --- |
| `rtl/` | Owner-written components and integrated core |
| `tb/` | cocotb component, instruction, and program tests |
| `scripts/run_regression.py` | Test scheduling, XML statistics, logs, seed forwarding |
| `docs/` | Implemented specification, architecture, verification, and debug evidence |
| `PROJECT_PLAN.md` | Scope, mentor rules, and staged v0.2+ plan |
| `PROGRESS.md` | Historical results and next-session handoff |
| `build/`, `reports/`, `waves/` | Reproducible generated artifacts, ignored by Git |

## Limitations and next session

- Only the 12 listed instruction types are supported; no jump, upper-immediate,
  subword memory, remaining branch, CSR, trap, interrupt, or privileged support.
- Memory has no ready/valid protocol or variable latency; tests supply reads
  before the committing clock edge and model writes at the edge.
- Tests use aligned instructions and word accesses. Alignment/access faults
  are not implemented; byte ordering is not verified by a word-only model.
- No whole-core ISA reference interpreter or formal verification is claimed.
- Programs are literal machine words in Python; assembler-to-image automation
  is deferred. No additional Python infrastructure exercise is required before
  instruction development resumes.
- Runner failure/exception paths have been reviewed, but have not all been
  exercised end-to-end. XML-reader failure/skipped counting was checked using
  a retained real failure report.

Stop this session at **DAY14 / v0.1**. Start a new session with
`PROJECT_PLAN.md`, `PROGRESS.md`, and `docs/specification.md` before beginning
v0.2 (logic/shifts/unsigned comparison), then v0.3 (branches) and v0.4
(byte/halfword memory). Core RTL and primary verification logic remain the
owner's work; auxiliary tooling is not a separate line-by-line course.

## References and attribution

Instruction semantics follow the
[RISC-V unprivileged ISA specification](https://docs.riscv.org/reference/isa/v20240411/unpriv/rv32.html).
This repository is an educational implementation, not an imported third-party
CPU core. Tool use and AI assistance with review, diagnostics, infrastructure,
and documentation are distinct from the owner's RTL and verification work.

Generated simulator output, reports, caches, and waveforms are ignored by Git.
