# RV32I Single-Cycle Processor

An owner-written educational 32-bit single-cycle CPU with automated
SystemVerilog/Verilator/cocotb verification.

**Current checkpoint: DAY17 / P1 v0.4 implementation checkpoint,
documented on 2026-09-20.**
This is a verified **RV32I-subset** core, not a complete RV32I implementation.
Version names identify development milestones; no Git tag or GitHub Release is
implied.

## Implemented scope

The integrated core supports these **33 instruction types**:

- ADD, ADDI, SUB
- AND, ANDI, OR, ORI, XOR, XORI
- SLT, SLTI, SLTU, SLTIU
- SLL, SLLI, SRL, SRLI, SRA, SRAI
- LB, LBU, LH, LHU, LW
- SB, SH, SW
- BEQ, BNE, BLT, BGE, BLTU, BGEU

`rtl/rv32i_core.sv` connects the PC, decoder, register file, immediate generator,
ALU, load/store interface, writeback selection, and branch comparison path. Instruction and
data memories are **external Python models**, not synthesized RAM modules.
The program tests fetch instruction words using the DUT's `current_pc`.

The v0.2 decoder selects XOR/XORI, unsigned comparisons, and register or
immediate shifts. The v0.3 decoder adds `branch_type` selection for BNE, BLT,
BGE, BLTU, and BGEU. The v0.4 decoder adds the byte/halfword load/store
encodings. Immediate shifts qualify the upper immediate bits according to the
RV32I encoding; ordinary I-type arithmetic continues to treat those bits as
immediate data.

The data interface uses a full 32-bit byte address and an aligned 32-bit
little-endian `data_read_data` word supplied by the external Python model.
`data_write_strb[3:0]` selects the written byte lanes, and
`data_write_data` places SB/SH payload bytes in those lanes. LB/LBU/SB may use
any byte address; LH/LHU/SH require address bit 0 to be zero; LW/SW require
address bits [1:0] to be zero. Misaligned halfword/word accesses that span two
aligned words are unsupported and unverified; no misalignment trap exists.
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
make regression SEED=20260920
```

`make regression` runs all seven test groups, reports case counts and failed
targets, and returns a nonzero status on a detected failure. It must run from
the repository root. `make test` alone still tests **only the full adder**.

## Verified v0.4 results

Full regression run on 2026-09-20:

| Target | cocotb cases | Passed | Failed | Skipped |
| --- | ---: | ---: | ---: | ---: |
| `test` | 1 | 1 | 0 | 0 |
| `test-alu` | 2 | 2 | 0 | 0 |
| `test-register-file` | 5 | 5 | 0 | 0 |
| `test-pc` | 3 | 3 | 0 | 0 |
| `test-immediate-generator` | 1 | 1 | 0 | 0 |
| `test-decoder` | 1 | 1 | 0 | 0 |
| `test-core` | 13 | 13 | 0 | 0 |
| **Total** | **26** | **26** | **0** | **0** |

The process exited with status 0. These are cocotb test-case counts, not
instruction-type counts, decoder-vector counts, dynamic-instruction counts, or
coverage percentages. Any simulator time printed for the core target is a
simulation detail, not an implementation-performance measurement.

The thirteen core cases check arithmetic, reset write blocking, LW/SW, the
subword load/store sequence, byte and halfword lane boundaries, BEQ, BNE,
signed BLT/BGE, unsigned BLTU/BGEU, PC-indexed straight-line execution, a
zero-initialized loop/store/load program, XOR/unsigned comparisons, and the
register/immediate shift group. The branch cases check `rf_we=0` and
`data_write_en=0`; the subword cases also check `data_write_strb=0` for loads,
lane-aligned write data, and byte preservation.
The shift case includes shift amount 31 and a register shift source of 32 to
verify RV32's low-five-bit rule; it also checks that these ALU instructions do
not assert `data_write_en`. The straight-line program finishes with
`x3=12`, `PC=12`.
The currently saved second program finishes with
`x1=x2=x3=memory[64]=0`, `PC=32`.

The decoder test is one cocotb case containing 37 legal and boundary vectors.
The integrated subset contains 33 instruction types. The current testbench and
regression runner do not instrument or report a dynamic-instruction execution
count; none is inferred from the 26 cocotb cases or 37 vectors.

The initial-5 loop previously passed with sum 15, but that variant was replaced
by initial 0 in the saved test. It is **historical evidence, not an additional
case in today's regression**. Initial 1 was explicitly skipped by the owner
and remains unverified. The reverse-direction and equality boundaries for the
four relational branches are also unverified; this is not exhaustive branch
acceptance. See the
[verification plan and evidence](docs/verification_plan.md).

Each group refreshes its XML in `reports/`. The runner saves stdout and stderr
to `reports/regression/<target>.log`, overwriting that target's previous log.
All seven latest logs confirm supplied cocotb seed `20260920`.
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
`test_core,test_lw_sw,test_beq,test_program`; the new subword cases are already
included through the existing `test_lw_sw` module. Wave generation is explicit, not
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
problems. The current v0.4 hierarchy contains **6142 generic cells**, including
1024 register-file enabled flip-flops, 32 PC flip-flops, and 172 decoder cells;
0 memory objects and 0 combinational latches were observed. These are
tool/run-specific structural counts, **not silicon area or Fmax**. No target
technology library, STA, post-layout timing, or gate-level equivalence result is
claimed. The v0.3 count of 5456 and v0.2 count of 5372 are historical.

The [synthesis record](docs/synthesis.md) contains the exact command, hierarchy,
warning disposition, and reproduction instructions. Generated evidence:

- `reports/lint/day13-core.log`
- `reports/synthesis/v0.4-core.log`
- `build/synthesis/v0.4-rv32i_core.v` (generated netlist, not hand-written source)

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

- Only the 33 listed instruction types are supported; no jump, upper-immediate,
  CSR, trap, interrupt, or privileged support.
- Memory has no ready/valid protocol or variable latency; tests supply reads
  before the committing clock edge and model writes at the edge.
- Byte accesses may use any byte address. Halfword accesses are supported only
  at even addresses, and word accesses only at four-byte-aligned addresses.
  Misaligned halfword/word accesses are not assembled/split across words and
  have no access-fault or misalignment exception; they are unsupported and
  unverified.
- No whole-core ISA reference interpreter or formal verification is claimed.
- Programs are literal machine words in Python; assembler-to-image automation
  is deferred. No additional Python infrastructure exercise is required before
  instruction development resumes.
- Runner failure/exception paths have been reviewed, but have not all been
  exercised end-to-end. XML-reader failure/skipped counting was checked using
  a retained real failure report.

This closeout stops at **DAY17 / v0.4**. Start the next session with
`PROJECT_PLAN.md`, `PROGRESS.md`, and `docs/specification.md` before planning
v0.5 U/J and jump support. Preserve the documented v0.3 branch gaps, initial-1
waiver, and v0.4 misalignment boundary. Core RTL and primary verification logic
remain the owner's work.

## References and attribution

Instruction semantics follow the
[RISC-V unprivileged ISA specification](https://docs.riscv.org/reference/isa/v20240411/unpriv/rv32.html).
This repository is an educational implementation, not an imported third-party
CPU core. Tool use and AI assistance with review, diagnostics, infrastructure,
and documentation are distinct from the owner's RTL and verification work.

Generated simulator output, reports, caches, and waveforms are ignored by Git.
