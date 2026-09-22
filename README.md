# RV32I Single-Cycle Processor

An owner-written educational 32-bit single-cycle CPU with automated
SystemVerilog/Verilator/cocotb verification.

**Current checkpoint: P1 v1.0 single-cycle engineering closeout, 2026-09-22.**
This is a verified **RV32I-subset** core, not a complete RV32I implementation.
Version names identify development milestones; no Git tag or GitHub Release is
implied.

## Implemented scope

The integrated core supports these **37 instruction types**:

- ADD, ADDI, SUB
- AND, ANDI, OR, ORI, XOR, XORI
- SLT, SLTI, SLTU, SLTIU
- SLL, SLLI, SRL, SRLI, SRA, SRAI
- LB, LBU, LH, LHU, LW
- SB, SH, SW
- BEQ, BNE, BLT, BGE, BLTU, BGEU
- LUI, AUIPC, JAL, JALR

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

The v0.5 path adds U- and J-type immediate generation, PC as an optional ALU
operand, `PC+4` link writeback, and direct/indirect jump target selection. JAL
uses `current_pc + Jimm`; JALR uses `(rs1 + Iimm) & ~1`. Instructions skipped
by a jump do not commit register or memory side effects in the directed test.

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
3.3.116, Yosys 0.33, and a project-local standalone OpenSTA 2.6.0 for basic
timing analysis. See [environment notes](docs/environment.md).
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
make regression SEED=20260922
```

`make regression` runs all seven test groups, reports case counts and failed
targets, and returns a nonzero status on a detected failure. It must run from
the repository root. `make test` alone still tests **only the full adder**.

## Verified v1.0 results

Full regression run on 2026-09-22:

| Target | cocotb cases | Passed | Failed | Skipped |
| --- | ---: | ---: | ---: | ---: |
| `test` | 1 | 1 | 0 | 0 |
| `test-alu` | 2 | 2 | 0 | 0 |
| `test-register-file` | 5 | 5 | 0 | 0 |
| `test-pc` | 3 | 3 | 0 | 0 |
| `test-immediate-generator` | 1 | 1 | 0 | 0 |
| `test-decoder` | 1 | 1 | 0 | 0 |
| `test-core` | 17 | 17 | 0 | 0 |
| **Total** | **30** | **30** | **0** | **0** |

The process exited with status 0. These are cocotb test-case counts, not
instruction-type counts, decoder-vector counts, dynamic-instruction counts, or
coverage percentages. Any simulator time printed for the core target is a
simulation detail, not an implementation-performance measurement.

The seventeen core cases check arithmetic, reset write blocking, LW/SW, the
subword load/store sequence, byte and halfword lane boundaries, BEQ, BNE,
signed BLT/BGE, unsigned BLTU/BGEU, PC-indexed straight-line execution, a
initial-zero and initial-one loop/store/load programs, XOR/unsigned comparisons, and the
register/immediate shift group, LUI/AUIPC, and JAL/JALR control flow. The
branch cases check `rf_we=0` and
`data_write_en=0`; the subword cases also check `data_write_strb=0` for loads,
lane-aligned write data, and byte preservation.
The shift case includes shift amount 31 and a register shift source of 32 to
verify RV32's low-five-bit rule; it also checks that these ALU instructions do
not assert `data_write_en`. The straight-line program finishes with
`x3=12`, `PC=12`.
The initial-zero program finishes with `x1=x2=x3=memory[64]=0`, `PC=32`.
The initial-one program executes the loop body once, then finishes with
`x1=0`, `x2=x3=memory[64]=1`, `PC=32`. Signed and unsigned relational-branch
tests now check both operand orders and equality, including taken/not-taken
PC results and inactive write enables.

The upper-immediate case checks LUI at PC 0 and AUIPC at PC 4, producing
`x1=0xabcde000` and `x2=0x12345004`. The jump case observes the PC path
`0 -> 4 -> 8 -> 16 -> 20 -> 40 -> 44`, checks JAL/JALR link values
`x1=12` and `x3=24`, verifies JALR clears an odd target's bit 0, and confirms
the skipped instructions leave their destination registers unchanged.
An additional core case checks a taken negative JAL offset and its `PC+4`
link without a memory-write side effect.

The immediate-generator test is one cocotb case containing 13 vectors. The
decoder test is one cocotb case containing 42 legal and boundary vectors.
The integrated subset contains 37 instruction types. The current testbench and
regression runner do not instrument or report a dynamic-instruction execution
count; none is inferred from the 30 cocotb cases, 13 immediate vectors, or 42
decoder vectors.

The initial-5 loop previously passed with sum 15, but that variant was replaced
by initial 0 in the saved test. It is **historical evidence, not an additional
case in today's regression**. The current directed branch and program cases
do not imply exhaustive ISA coverage. See the
[verification plan and evidence](docs/verification_plan.md).

Each group refreshes its XML in `reports/`. The runner saves stdout and stderr
to `reports/regression/<target>.log`, overwriting that target's previous log.
All seven latest logs confirm supplied cocotb seed `20260922`.
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
the I/S/B/U/J immediate generator does not consume opcode bits `instr[6:0]`.
No warning class was disabled. `-Wno-fatal` keeps warnings visible while
allowing the command to complete; status 0 does not mean warning-free RTL.

Generic Yosys synthesis and `check -assert` succeeded with 0 reported structural
problems. The v0.5/v1.0 RTL hierarchy contains **6642 generic cells**, including
1024 register-file enabled flip-flops, 32 PC flip-flops, 206 decoder cells, and
216 immediate-generator cells;
0 memory objects and 0 combinational latches were observed. These are
tool/run-specific structural counts, **not silicon area or Fmax**. The v0.4
count of 6142, v0.3 count of 5456, and v0.2 count of 5372 are historical.

A separate Nangate45-typical mapping produced 6267 library cells and passed
Yosys `check -assert`. Basic pre-layout, core-only STA at an **assumed** 10 ns
period reported +5.202 ns worst setup slack (WNS/TNS 0). The path is
`instr[21]` to `data_write_data[10]`. Fifteen maximum-slew violations remain;
external memory timing, placement, routing, and other corners were not
analyzed. This is neither timing closure nor a measured CPU Fmax. The
[timing analysis record](timing/README.md) gives tools, constraints, warnings,
and reproduction details. No gate-level functional equivalence result is claimed.

The [synthesis record](docs/synthesis.md) contains the exact command, hierarchy,
warning disposition, and reproduction instructions. Generated evidence:

- `reports/lint/day13-core.log`
- `reports/synthesis/v0.5-core.log`
- `build/synthesis/v0.5-rv32i_core.v` (generated netlist, not hand-written source)
- `reports/timing/core_setup_nangate45_typ_10ns.txt` (tracked STA output)

## Repository map

| Path | Purpose |
| --- | --- |
| `rtl/` | Owner-written components and integrated core |
| `tb/` | cocotb component, instruction, and program tests |
| `scripts/run_regression.py` | Test scheduling, XML statistics, logs, seed forwarding |
| `docs/` | Implemented specification, architecture, verification, and debug evidence |
| `timing/` | Reproducible core-only STA constraints, script, and interpretation |
| `PROJECT_PLAN.md` | Scope, mentor rules, and staged v1.0+ plan |
| `PROGRESS.md` | Historical results and next-session handoff |
| `build/`, `reports/`, `waves/` | Generated artifacts; only the v1.0 STA report is retained in Git |

## Limitations and next session

- Only the 37 listed instruction types are supported; there is no FENCE,
  system/CSR, trap, interrupt, or privileged support.
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
- JALR target bit 0 clearing is verified. Instruction-address misalignment
  exceptions are not implemented; targets with bit 1 set are unsupported and
  unverified under the current four-byte-aligned instruction-memory contract.

This engineering closeout completes **P1 v1.0** within the documented external
memory and timing limitations. Begin v2.0 by defining the five pipeline stage
contracts and pipeline registers while preserving this single-cycle baseline.
Core RTL and the principal verification architecture remain the owner's work.

## References and attribution

Instruction semantics follow the
[RISC-V unprivileged ISA specification](https://docs.riscv.org/reference/isa/v20240411/unpriv/rv32.html).
This repository is an educational implementation, not an imported third-party
CPU core. Tool use and AI assistance with review, diagnostics, infrastructure,
and documentation are distinct from the owner's RTL and verification work.

Generated simulator output, caches, and waveforms are ignored by Git. The
v1.0 core-only STA report is the one intentionally tracked report.
