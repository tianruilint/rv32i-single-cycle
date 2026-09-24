# Verification

## Regression

Recorded run: 2026-09-24, Verilator 5.050 and cocotb 2.0.1.

```sh
make regression SEED=20260924
```

All 17 targets passed: **74 cases, 74 passed, 0 failed, 0 skipped**.

| Make target | XML under build/reports/ | Cases |
| --- | --- | ---: |
| `test` | full_adder.xml | 1 |
| `test-alu` | alu.xml | 2 |
| `test-register-file` | register_file.xml | 5 |
| `test-pc` | pc.xml | 3 |
| `test-immediate-generator` | immediate_generator.xml | 1 |
| `test-decoder` | decoder.xml | 1 |
| `test-core` | core.xml | 17 |
| `test-single-cycle-benchmark` | single_cycle_benchmark.xml | 3 |
| `test-pipeline-frontend` | pipeline_frontend.xml | 1 |
| `test-pipeline-id` | pipeline_id.xml | 5 |
| `test-pipeline-id-ex` | pipeline_id_ex.xml | 1 |
| `test-pipeline-ex` | pipeline_ex.xml | 13 |
| `test-pipeline-ex-mem` | pipeline_ex_mem.xml | 5 |
| `test-pipeline-hazard` | pipeline_hazard.xml | 2 |
| `test-pipeline-mem` | pipeline_mem.xml | 3 |
| `test-pipeline-mem-wb` | pipeline_mem_wb.xml | 1 |
| `test-pipeline-core` | pipeline_core.xml | 10 |

The ALU's two cases contain 19 directed and 2,000 seeded input vectors.
The immediate and decoder cases contain 13 and 42 vectors respectively.
These vector counts, the 37 supported instruction types, and dynamic retired
instruction counts are separate from the 74 cocotb cases.

## Test structure

| Layer | Source | Checks |
| --- | --- | --- |
| Shared components | `test_alu.py`, `test_register_file.py`, `test_pc.py`, `test_immediate_generator.py`, `test_decoder.py` | Independent ALU expectations, x0/write enable, PC control, I/S/B/U/J fields, decode qualification |
| Single-cycle core | `test_core.py`, `test_lw_sw.py`, `test_beq.py`, `test_program.py` | Register/PC results, memory lanes, branch outcomes, jump links, reset |
| Pipeline units | `test_pipeline_*.py` excluding `test_pipeline_core.py` | Stage-register reset/bubbles, bypass/forwarding priority, hazards, EX redirects, MEM lane selection, write gating |
| Pipeline programs | `test_pipeline_core.py` | Register and byte-memory results, retirement-PC traces, stalls, redirects, and in-flight reset |
| Matched benchmarks | `test_single_cycle_benchmark.py`, `test_pipeline_core.py` | Same machine-code programs on both cores, final state and cycle counts |

The integrated pipeline cases cover:

- EX/MEM and MEM/WB forwarding, including LUI and link values.
- Load-to-ALU, load-to-store, and load-to-branch dependencies.
- Taken branch, JAL, and JALR redirects; wrong-path stores/register writes.
- All six branch conditions across unit and program tests.
- Byte/halfword sign and zero extension, byte preservation, x0, and reset while
  a store is in flight.
- Signed/unsigned comparisons, logical/arithmetic shifts, upper immediates,
  and a backward-branch loop.

The single-cycle cases additionally check each relational branch with both
operand orders and equality, register shifts using 31 and 32, a negative JAL
target, and loop counters initialized to zero and one. Passing these directed
cases does not establish exhaustive instruction or input coverage.

## Memory and sampling

The testbench supplies `instr` for the fetch PC, waits for combinational
outputs to settle, and supplies an aligned 32-bit little-endian read word.
Before the rising edge it captures store address, data, enable, and strobes.
At the edge the Python model writes only selected bytes; after settling it
checks architectural state. Simulator address values are converted with
`int(...)` before dictionary access.

Each independent case establishes its own reset, inputs, and register
preconditions. The register-file array has no reset. Pipeline program and
benchmark memory helpers return zero for absent bytes; that is a testbench
choice, not an architectural guarantee about uninitialized memory.

Pipeline program fetch uses an ADDI x0,x0,0 outside the supplied program while
the pipeline drains. Completion is the retirement of a designated terminal
instruction; the CPU itself has no halt port or halt instruction.

## Cycle and CPI measurements

The matched programs are in `test_single_cycle_benchmark.py` and
`test_pipeline_core.py`. They check the same register/memory results.
The single-cycle harness counts committing edges through its terminal
instruction. The pipeline harness counts from the first non-reset edge
through retirement of the terminal instruction and checks `retired_count`
against the observed retirement trace.

| Program | Retired instructions | Single-cycle cycles | Pipeline cycles | Load-use stalls | Taken redirects |
| --- | ---: | ---: | ---: | ---: | ---: |
| ALU dependencies | 8 | 8 | 12 | 0 | 0 |
| Subword memory | 10 | 10 | 16 | 2 | 0 |
| Loop sum 1..5 | 22 | 22 | 35 | 1 | 4 |

For these programs the pipeline counts satisfy
`cycles = retired + 4 + load_use_stalls + 2 * taken_redirects`.
The four extra cycles fill the pipeline. Each load-use hazard inserts one
bubble, and each taken EX redirect discards two younger slots.
CPI is `cycles / retired`: 1.5000, 1.6000, and 1.5909 for the pipeline;
1.0000 for the single-cycle core.

These measurements use zero-wait memory. They do not measure execution time
at a physically achievable clock frequency.

## Reports and reproduction

Each target overwrites its XML under `build/reports/`. The runner combines
stdout/stderr in `build/reports/regression/<target>.log`, also overwritten per
target. Benchmark lines in the two core benchmark logs contain cycles,
retired instructions, and CPI.

A nonzero Make return code fails the target. After a successful command the
runner rejects missing/malformed XML, zero cases, failures/errors, and skipped
cases. Printed totals include only parsed reports, so the failed-target list
and process status determine whether the complete run passed.

The runner supplies `COCOTB_RANDOM_SEED` through `SEED`. The ALU vector
generator separately uses `random.Random(20260906)`. Source, tool versions,
and test selection are also required to reproduce results.

```sh
make test-pipeline-core
make test-single-cycle-benchmark
make waves-core
gtkwave build/waves/core.fst
```

To isolate the single-cycle zero-initialized loop:

```sh
make waves-core CORE_TEST_MODULES=test_program COCOTB_TEST_FILTER='test_program2$'
```

Waveforms are generated only by explicit waveform targets, saved under
`build/waves/`, and ignored by Git. Wave targets share the temporary
`dump.fst` name and must run serially. The count of saved traces is not a
test-coverage metric.

The output layout is:

| Path | Contents |
| --- | --- |
| `build/reports/*.xml` | Latest cocotb results per target |
| `build/reports/regression/*.log` | Latest regression stdout/stderr per target |
| `build/waves/*.fst` | Waveforms generated on request |
| `reports/timing/*.txt` | Two saved STA analysis reports |

`make clean` clears generated simulation products, test reports, and waveforms.
It preserves `build/timing/` dependencies and the saved `reports/timing/`
evidence. STA constraints and interpretation are in [synthesis.md](synthesis.md).

## Verification limits

There is no independent whole-core ISA interpreter, randomized differential
test, formal proof, gate-level equivalence result, or automated coverage
percentage. The ALU has an independent Python reference; program checks use
directed expected architectural values and selected retirement traces.

Unsupported/misaligned instructions, every reset interaction, variable-latency
memory, and all runner error paths are not exhaustively tested. Some assertions
inspect internal register storage and therefore depend on the RTL hierarchy.
