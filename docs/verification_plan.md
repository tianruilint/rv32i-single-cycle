# v0.5 Verification Plan and Evidence

Checkpoint: 2026-09-21. Results below are observed, not proposed coverage.

## Layers and responsibilities

- `tb/test_alu.py`: independent Python arithmetic/logic expectations, including
  directed and seeded vectors, for the ALU component.
- Other component tests check register-file, PC, immediate, and decoder behavior.
- Core tests drive instructions and inspect architectural register/PC/memory
  results, with some control and internal-hierarchy checks.
- `tb/test_program.py` adds PC-indexed instruction fetch and external memory
  behavior; its Python loop simulates cycles rather than computing the DUT's sum.
- `scripts/run_regression.py` schedules existing tests and collects results. It
  is not a CPU reference model or a replacement for the testbenches.

There is no independent whole-core ISA interpreter, formal proof, or automated
functional/line/branch coverage measurement in this checkpoint.

## Latest full regression

Executed from the repository root in WSL:

```sh
make regression SEED=20260921
```

| Target | Test source | XML under reports/ | Cases / pass / fail / skip |
| --- | --- | --- | --- |
| test | test_full_adder.py | full_adder.xml | 1 / 1 / 0 / 0 |
| test-alu | test_alu.py | alu.xml | 2 / 2 / 0 / 0 |
| test-register-file | test_register_file.py | register_file.xml | 5 / 5 / 0 / 0 |
| test-pc | test_pc.py | pc.xml | 3 / 3 / 0 / 0 |
| test-immediate-generator | test_immediate_generator.py | immediate_generator.xml | 1 / 1 / 0 / 0 |
| test-decoder | test_decoder.py | decoder.xml | 1 / 1 / 0 / 0 |
| test-core | test_core.py, test_lw_sw.py, test_beq.py, test_program.py | core.xml | 15 / 15 / 0 / 0 |
| **Total** | | | **28 / 28 / 0 / 0** |

Exit status: 0. Test count is distinct from vector count, instruction-type
count, and dynamic-instruction count.
The ALU's 19 directed vectors and 2000 seeded vectors are grouped into two
cocotb cases, not 2019 cases. The immediate-generator test is one case with
13 vectors. The decoder test is one cocotb case containing 42 legal and
boundary instruction vectors. The implemented subset contains 37 instruction
types. The current testbench and regression runner do not
instrument or report a dynamic-instruction execution count; no dynamic total
is inferred from the case or vector counts.

`Makefile` keeps `CORE_TEST_MODULES := test_core,test_lw_sw,test_beq,test_program`;
the two new subword cases are discovered through the existing `test_lw_sw`
module, so no redundant regression entry was added.
The two v0.5 core cases are in the existing `test_core.py` module and therefore
also require no Makefile module-list change.

## Core cases and architectural expectations

| Case | Main observation |
| --- | --- |
| test_arithmetic_chain | ADD/ADDI/SUB/AND/ANDI/OR/ORI/SLT/SLTI results, PC progression, x0 protection |
| test_reset_blocks_register_write | Existing x13 value survives an attempted write while reset forces PC=0 |
| test_lw_sw | Store 42 at byte address 72 and load 42 into x3 |
| test_subword_load_store | LB/LBU/LH/LHU/SB/SH behavior, sign/zero extension, lane-aligned stores, and byte preservation |
| test_subword_lane_boundaries | All four byte lanes, both aligned halfword lanes, and load-side lane selection |
| test_beq | Taken +8, not taken, negative -12 branch; a taken BEQ's write enables are checked inactive |
| test_bne | Equal operands not taken and unequal operands taken; both branch write enables inactive |
| test_blt_bge | Signed -1 versus 1: BLT taken and BGE not-taken; both branch write enables inactive |
| test_bltu_bgeu | Unsigned 0xffffffff versus 1: BLTU not-taken and BGEU taken; both branch write enables inactive |
| test_program | Fetch at PC 0/4/8; final x3=12, PC=12 |
| test_program2 | Current initial-0 program exits loop, stores/loads 0 at byte address 64, ends at PC=32 |
| test_xor_sltu_instrs | XOR/XORI and SLTU/SLTIU results, including unsigned ordering and sign-extended immediate behavior |
| test_shift_instrs | SLL/SLLI, SRL/SRLI, SRA/SRAI; shift amount 31, register source 32, final PC=52, and `data_write_en=0` |
| test_upper_immediate_instrs | LUI at PC 0 and AUIPC at PC 4; expected x1/x2 values and no memory-write side effect |
| test_jal_jalr_control_flow | PC path `0,4,8,16,20,40`, JAL/JALR links, odd-target bit-0 clearing, skipped destinations unchanged, and no memory write |

This exercises the 37 supported instruction types, but does not prove all their
input combinations or all side effects under every condition. In particular,
not every unsupported encoding, reset/memory interaction, or misaligned
halfword/word access is tested. The relational branch reverse directions and
equality boundaries are not tested. The integrated jump test uses one positive
JAL offset and one forward JALR target. A negative J immediate is checked at
the component level, but a negative taken JAL, target bit 1 behavior, and an
instruction-address-misalignment exception are not integrated/verified.
Current tests inspect internal register
storage for some assertions; that hierarchy is a test dependency, not a stable
external hardware interface.

## Loop variants: current versus historical

| Initial counter | Evidence | Current default case? |
| --- | --- | --- |
| 5 | Historical DAY11 run: x1=0, x2=x3=memory[64]=15, PC=32; six-case report passed, 586 ns | No; later replaced by initial 0 |
| 0 | Latest fifteen-case core regression passes; x1=x2=x3=memory[64]=0, PC=32 | Yes |
| 1 | Explicitly skipped by the owner | No; unverified |

Initial-5 historical XML: `reports/day11-program2-review2/core.xml`.
Initial-zero waveform: `waves/day11-zero-wave/core.fst` (1931 bytes), with
`reports/day11-zero-wave/core-waves.xml` reporting 1/1 selected case, 51 ns.
The historical PC trace was inspected as `0 -> 4 -> 8 -> 24 -> 28 -> 32`;
the loop body is bypassed before SW and LW at address 64.

The current regression does not execute the nonzero loop body in test_program2.
If retaining both 0 and 5 as permanent tests becomes desired, obtain agreement
and have the owner add that case; do not silently change the baseline during
documentation work. Do not add initial 1 after it was explicitly waived.

## Memory-model ordering

The test supplies `instr`, lets combinational signals settle, converts DUT
addresses with `int(...)`, and supplies aligned little-endian load data before
the rising edge. It captures store address/data/enable/strobe for the
committing edge and updates only the selected bytes in the Python dictionary.
After the edge, it allows signal updates to settle before inspecting
architectural state. `read_word` assembles an aligned 32-bit word and
`apply_write` preserves bytes whose strobe bits are zero.

No real SRAM, memory latency, ready/valid handshake, cross-word assembly/split,
or alignment exception is modeled. Literal machine words are embedded in
Python; no assembler/program-image build is run. LB/LBU/SB may use any byte
address; LH/LHU/SH require bit 0 clear; LW/SW require bits [1:0] clear.

The new `test_subword_lane_boundaries` case initially omitted its own
clock/reset/x1/x2 initialization. The simulator shut down prematurely and
produced cascading zero-nanosecond failures. The case now initializes its own
clock, reset, instruction input, load input, and address registers, so it runs
independently of the other `test_lw_sw` cases.

## Runner behavior and limitations

Each Make target removes its previous XML before its normal simulation. The
runner handles a nonzero Make return code as target failure without parsing
that target's XML. If Make succeeds, it parses testcase elements and rejects
missing/malformed XML, zero cases, failures/errors, or skipped cases. It
continues to the remaining targets for these handled failures.

Consequently, printed case totals cover successfully parsed reports only;
`Failed targets` and the exit status determine whole-run success. A failed
build with no parsed report must not be interpreted as zero failures overall.

The full success path was executed. The XML reader was also checked against
the retained real `LogicArray` failure report and returned `(2, 1, 1)` for
total/failed/skipped. The skipped case in that historical report was excluded
by test selection; it was not the waived initial-1 boundary test.
The complete runner error-path matrix has not been dynamically validated.

Logs: `reports/regression/<target>.log`, stdout and stderr combined, overwritten
per target. XML remains directly under `reports/`. No per-run manifest or
immutable run archive is generated. A failure opening a log or launching Make
is not handled by the report-parsing exception handler.

`--seed` is parsed by the runner and passed through the subprocess environment
as `COCOTB_RANDOM_SEED`. All seven current logs confirm `20260921`.
`random.Random(20260906)` in the ALU test is independently seeded. Reproduction
also requires the same source, test selection, and compatible toolchain; the
seed alone is not a complete environment record.

## Waveform reproduction

```sh
make waves-core
gtkwave waves/core.fst
```

To focus on the currently saved zero-initialized program:

```sh
make waves-core CORE_TEST_MODULES=test_program COCOTB_TEST_FILTER=test_program2
```

These are reproduction instructions; the v0.5 closeout did not rerun the
historical zero waveform or open GTKWave. Automatic waveform generation on
failure is not implemented. Wave targets share `dump.fst`; do not run them
concurrently.

## Acceptance for subsequent instruction groups

For each agreed group: define semantics/encoding and hardware corner cases,
let the owner implement RTL and principal tests, run directed and useful
boundary tests, inspect any real failure, then rerun the existing regression.
The v0.4 memory contract and v0.5 upper-immediate/jump contract are implemented
and tested at their documented boundaries. Only claim newly implemented
instruction support after those tests execute. Formal verification, exhaustive coverage, a dynamic
instruction counter, and a whole-core reference model remain separate future
decisions, not existing results. The next milestone is v1.0 stabilization,
including review of remaining correctness gaps and a real basic timing-analysis
setup.
