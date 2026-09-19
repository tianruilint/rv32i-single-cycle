# RV32I Single-Cycle Processor

An owner-written educational 32-bit single-cycle CPU with automated
SystemVerilog/Verilator/cocotb verification.

**Current checkpoint: DAY16 / P1 v0.3 implementation checkpoint,
documented on 2026-09-19.**
This is a verified **RV32I-subset** core, not a complete RV32I implementation.
Version names identify development milestones; no Git tag or GitHub Release is
implied.

## Implemented scope

The integrated core supports these **27 instruction types**:

- ADD, ADDI, SUB
- AND, ANDI, OR, ORI, XOR, XORI
- SLT, SLTI, SLTU, SLTIU
- SLL, SLLI, SRL, SRLI, SRA, SRAI
- LW, SW
- BEQ, BNE, BLT, BGE, BLTU, BGEU

`rtl/rv32i_core.sv` connects the PC, decoder, register file, immediate generator,
ALU, load/store interface, writeback selection, and branch comparison path. Instruction and
data memories are **external Python models**, not synthesized RAM modules.
The program tests fetch instruction words using the DUT's `current_pc`.

The v0.2 decoder selects XOR/XORI, unsigned comparisons, and register or
immediate shifts. The v0.3 decoder adds `branch_type` selection for BNE, BLT,
BGE, BLTU, and BGEU. Immediate shifts qualify the upper immediate bits
according to the RV32I encoding; ordinary I-type arithmetic continues to treat
those bits as immediate data.
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
make regression SEED=20260919
```

`make regression` runs all seven test groups, reports case counts and failed
targets, and returns a nonzero status on a detected failure. It must run from
the repository root. `make test` alone still tests **only the full adder**.

## Verified v0.3 results

Full regression run on 2026-09-19:

| Target | cocotb cases | Passed | Failed | Skipped |
| --- | ---: | ---: | ---: | ---: |
| `test` | 1 | 1 | 0 | 0 |
| `test-alu` | 2 | 2 | 0 | 0 |
| `test-register-file` | 5 | 5 | 0 | 0 |
| `test-pc` | 3 | 3 | 0 | 0 |
| `test-immediate-generator` | 1 | 1 | 0 | 0 |
| `test-decoder` | 1 | 1 | 0 | 0 |
| `test-core` | 11 | 11 | 0 | 0 |
| **Total** | **24** | **24** | **0** | **0** |

The process exited with status 0. These are test-case counts, not instruction
counts or coverage percentages. The current core target simulated for 781 ns;
this is not an implementation-performance measurement.

The eleven core cases check arithmetic, reset write blocking, LW/SW, BEQ,
BNE, signed BLT/BGE, unsigned BLTU/BGEU, PC-indexed straight-line execution,
a zero-initialized loop/store/load program, XOR/unsigned comparisons, and the
register/immediate shift group. The new branch cases check `rf_we=0` and
`data_write_en=0`.
The shift case includes shift amount 31 and a register shift source of 32 to
verify RV32's low-five-bit rule; it also checks that these ALU instructions do
not assert `data_write_en`. The straight-line program finishes with
`x3=12`, `PC=12`.
The currently saved second program finishes with
`x1=x2=x3=memory[64]=0`, `PC=32`.

The initial-5 loop previously passed with sum 15, but that variant was replaced
by initial 0 in the saved test. It is **historical evidence, not an additional
case in today's regression**. Initial 1 was explicitly skipped by the owner
and remains unverified. The reverse-direction and equality boundaries for the
four relational branches are also unverified; this is not exhaustive branch
acceptance. See the
[verification plan and evidence](docs/verification_plan.md).

Each group refreshes its XML in `reports/`. The runner saves stdout and stderr
to `reports/regression/<target>.log`, overwriting that target's previous log.
All seven latest logs confirm supplied cocotb seed `20260919`.
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
problems. The current v0.3 hierarchy contains **5456 generic cells**, including
1024 register-file enabled flip-flops, 32 PC flip-flops, and 169 decoder cells.
No latch was inferred from the combinational processes. These are tool/run-specific
structural counts, **not silicon area or Fmax**. No target technology library,
STA, post-layout timing, or gate-level equivalence result is claimed. The v0.2
count of 5372 is historical and is not the v0.3 result.

The [synthesis record](docs/synthesis.md) contains the exact command, hierarchy,
warning disposition, and reproduction instructions. Generated evidence:

- `reports/lint/day13-core.log`
- `reports/synthesis/v0.3-core.log`
- `build/synthesis/v0.3-rv32i_core.v` (generated netlist, not hand-written source)

## Repository map

| Path | Purpose |
| --- | --- |
| `rtl/` | Owner-written components and integrated core |
| `tb/` | cocotb component, instruction, and program tests |
| `scripts/run_regression.py` | Test scheduling, XML statistics, logs, seed forwarding |
| `docs/` | Implemented specification, architecture, verification, and debug evidence |
| `PROJECT_PLAN.md` | Scope, mentor rules, and staged v0.3+ plan |
| `PROGRESS.md` | Historical results and next-session handoff |
| `build/`, `reports/`, `waves/` | Reproducible generated artifacts, ignored by Git |

## Limitations and next session

- Only the 27 listed instruction types are supported; no jump, upper-immediate,
  subword memory, CSR, trap, interrupt, or privileged support.
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

This closeout stops at **DAY16 / v0.3**. Start the next session with
`PROJECT_PLAN.md`, `PROGRESS.md`, and `docs/specification.md` before planning
v0.4 byte/halfword memory. First settle the byte-addressed little-endian
contract, low-bit lane selection, write strobes, byte preservation,
sign/zero extension, and alignment/access exceptions; only then change the
core and Python memory model. Core RTL and primary verification logic remain
the owner's work.

## References and attribution

Instruction semantics follow the
[RISC-V unprivileged ISA specification](https://docs.riscv.org/reference/isa/v20240411/unpriv/rv32.html).
This repository is an educational implementation, not an imported third-party
CPU core. Tool use and AI assistance with review, diagnostics, infrastructure,
and documentation are distinct from the owner's RTL and verification work.

Generated simulator output, reports, caches, and waveforms are ignored by Git.
