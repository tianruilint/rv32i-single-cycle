# Bug Diary

This diary records actual development failures and intermediate review findings,
not hypothetical bugs or invented fault-injection evidence. Updated for DAY17
on 2026-09-20.

## DAY11 — Simulator value used as a Python dictionary key

- **Recorded:** DAY11 work; summarized at DAY14 on 2026-09-15.
- **Symptom:** `test_program2` raised
  `TypeError: unhashable type: 'LogicArray'` at simulated time 241 ns.
- **Evidence:** `reports/day11-program2-review1/core.xml` contains the TypeError
  failure. Its two listed cases consist of one failed selected case and one
  skipped case; that skipped case is unrelated to the waived initial-1 test.
- **Root cause:** a simulator address value was passed directly to the Python
  memory dictionary instead of converting it to a hashable integer address.
- **Owner fix:** use `int(dut.data_addr.value)` for dictionary addressing. The
  corrected model also explicitly drives `dut.data_read_data` before LW's
  clock edge and applies SW using the captured address/data/enable.
- **Validation:** the initial-5 program subsequently passed with sum 15; the
  retained report `reports/day11-program2-review2/core.xml` has six passing
  cases. The saved initial-0 variant also passes in the latest 21-case regression.
- **Waveform boundary:** the retained, inspected waveform belongs to the later
  initial-zero run; no failing-waveform observation is asserted for this error.
- **Lesson:** a DUT value handle/LogicArray is not a Python integer, and a value
  stored in a Python local variable does not drive a DUT input. Type conversion
  and pre-edge data timing are both part of a correct external memory model.

Earlier register-enable, writeback-direction, LW/SW, and BEQ integration issues
are recorded in the DAY10 section of `PROGRESS.md`. Their history is preserved
there without manufacturing missing failure logs or waveform measurements.

## DAY15 — Immediate-shift upper-field qualification

- **Recorded:** 2026-09-16 during the v0.2 decoder extension.
- **Symptom:** the targeted decoder test reported an unsupported immediate-shift
  encoding as active. In particular, an SLLI-shaped instruction with a nonzero
  shift upper field was accepted instead of retaining the inactive safe default.
- **Root cause:** the ordinary I-type decode path correctly treats bits
  `[31:25]` as immediate data, but the three RV32 shift-immediate forms are a
  special encoding. The first v0.2 implementation did not qualify that field
  before enabling SLLI/SRLI/SRAI.
- **Owner fix:** require `0000000` for SLLI and SRLI, and `0100000` for SRAI;
  add legal and illegal shift-encoding vectors to `tb/test_decoder.py`.
- **Validation:** `make regression SEED=20260916` passed 21/21 cases, including
  the decoder and eight core cases. `make lint-core` exited 0.
- **Lesson:** field names such as `funct7` describe a bit position only in the
  relevant instruction format. Ordinary I-type upper bits are data; shift
  immediates are the deliberate constrained exception.

## Template for future real issues

## DAY16 — v0.3 branch-extension review findings

These were intermediate issues found and corrected during the owner-written
branch extension. They are retained as development history; the current
working files passed lint, regression, and synthesis.

- A temporary `NOEN` spelling typo was corrected to `NONE` for the default
  branch type.
- A BEQ expected tuple was temporarily duplicated while extending the decoder
  vector list. The current list has one BEQ vector and 29 total vectors.
- A trailing-comma/`NULLPORT` interface issue appeared while adding the new
  decoder output. The `branch_type` port is now declared and connected through
  the decoder/core interface; current Verilator and Yosys runs report no such
  source error.
- The core branch selector was completed only after the decoder `branch_type`
  signal was wired into the core. The final path selects equality, signed
  less-than, unsigned less-than, or the inverse relation.

Coverage boundary, not a resolved bug: the current directed tests do not cover
the reverse ordering or equality boundary for BLT/BGE/BLTU/BGEU. Initial-1
program behavior remains explicitly waived by the owner.

## DAY17 — Subword boundary test initialization

- **Recorded:** 2026-09-20 during the v0.4 subword memory verification.
- **Symptom:** `test_subword_lane_boundaries` initially omitted its own clock,
  reset, and x1/x2 setup. The simulator shut down prematurely and the selected
  run produced cascading zero-nanosecond failures.
- **Root cause:** the new cocotb case depended on initialization performed by a
  different test instead of establishing its own DUT and register state.
- **Fix:** initialize the clock, reset, instruction input, load-data input, and
  address registers inside the case before driving the store/load boundary
  vectors.
- **Validation:** the current case runs independently as part of the existing
  `test_lw_sw` module; the v0.4 core target and full regression pass without a
  skipped case.
- **Lesson:** every cocotb case that can run independently must own its clock,
  reset, inputs, and architectural preconditions. A simulator shutdown can
  otherwise create misleading zero-time follow-on failures.

Copy this template when a new issue is actually encountered.

## YYYY-MM-DD — Short issue title

- **Date:** YYYY-MM-DD
- **Symptom:** What was observed, including the failing command or test.
- **Root cause:** Why the problem occurred.
- **Fix:** What changed to correct it.
- **Lesson learned:** What should be remembered or improved next time.
