# v0.1 Verification Plan and Evidence

Checkpoint: 2026-09-15. Results below are observed, not proposed coverage.

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
make regression SEED=20260915
```

| Target | Test source | XML under reports/ | Cases / pass / fail / skip |
| --- | --- | --- | --- |
| test | test_full_adder.py | full_adder.xml | 1 / 1 / 0 / 0 |
| test-alu | test_alu.py | alu.xml | 2 / 2 / 0 / 0 |
| test-register-file | test_register_file.py | register_file.xml | 5 / 5 / 0 / 0 |
| test-pc | test_pc.py | pc.xml | 3 / 3 / 0 / 0 |
| test-immediate-generator | test_immediate_generator.py | immediate_generator.xml | 1 / 1 / 0 / 0 |
| test-decoder | test_decoder.py | decoder.xml | 1 / 1 / 0 / 0 |
| test-core | test_core.py, test_lw_sw.py, test_beq.py, test_program.py | core.xml | 6 / 6 / 0 / 0 |
| **Total** | | | **19 / 19 / 0 / 0** |

Exit status: 0. The core suite's simulated duration is 386 ns. Test count is
distinct from vector count and instruction count. The ALU's 19 directed vectors
and 2000 seeded vectors are grouped into two cocotb cases, not 2019 cases.

## Core cases and architectural expectations

| Case | Main observation |
| --- | --- |
| test_arithmetic_chain | ADD/ADDI/SUB/AND/ANDI/OR/ORI/SLT/SLTI results, PC progression, x0 protection |
| test_reset_blocks_register_write | Existing x13 value survives an attempted write while reset forces PC=0 |
| test_lw_sw | Store 42 at byte address 72 and load 42 into x3 |
| test_beq | Taken +8, not taken, negative -12 branch; a taken BEQ's write enables are checked inactive |
| test_program | Fetch at PC 0/4/8; final x3=12, PC=12 |
| test_program2 | Current initial-0 program exits loop, stores/loads 0 at byte address 64, ends at PC=32 |

This exercises the 12 supported instruction types, but does not prove all their
input combinations or all side effects under every condition. In particular,
not every unsupported encoding, reset/memory interaction, or alignment case is
tested. Current tests inspect internal register storage for some assertions;
that hierarchy is a test dependency, not a stable external hardware interface.

## Loop variants: current versus historical

| Initial counter | Evidence | Current default case? |
| --- | --- | --- |
| 5 | Historical DAY11 run: x1=0, x2=x3=memory[64]=15, PC=32; six-case report passed, 586 ns | No; later replaced by initial 0 |
| 0 | Latest six-case core regression passes; x1=x2=x3=memory[64]=0, PC=32 | Yes |
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
addresses with `int(...)`, and supplies load data before the rising edge.
It captures SW address/data/enable for the committing edge and updates the
Python dictionary for that transaction. After the edge, it allows signal
updates to settle before inspecting architectural state.

No real SRAM, memory latency, or ready/valid handshake is modeled. Literal
machine words are embedded in Python; no assembler/program-image build is run.

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
as `COCOTB_RANDOM_SEED`. All seven current logs confirm `20260915`.
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

These are reproduction instructions; DAY14 did not rerun the historical zero
waveform or open GTKWave. Automatic waveform generation on failure is not
implemented. Wave targets share `dump.fst`; do not run them concurrently.

## Acceptance for subsequent instruction groups

For each agreed group: define semantics/encoding and hardware corner cases,
let the owner implement RTL and principal tests, run directed and useful
boundary tests, inspect any real failure, then rerun the existing regression.
Only claim newly implemented instruction support after those tests execute.
Formal verification, exhaustive coverage, and a whole-core reference model
remain separate future decisions, not existing results.
