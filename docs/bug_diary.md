# Bug Diary

This diary records actual development failures and intermediate review findings,
not hypothetical bugs or invented fault-injection evidence. Historical DAY18
entries were reviewed at the v1.0 closeout on 2026-09-22; the added directed
cases found no new RTL bug.

## DAY11 — Simulator value used as a Python dictionary key

- **Recorded:** DAY11 work; summarized at DAY14 on 2026-09-15.
- **Symptom:** `test_program2` raised
  `TypeError: unhashable type: 'LogicArray'` at simulated time 241 ns.
- **Evidence:** `reports/day11-program2-review1/core.xml` contains the TypeError
  failure. Its two listed cases consist of one failed selected case and one
  skipped case from test selection.
- **Root cause:** a simulator address value was passed directly to the Python
  memory dictionary instead of converting it to a hashable integer address.
- **Owner fix:** use `int(dut.data_addr.value)` for dictionary addressing. The
  corrected model also explicitly drives `dut.data_read_data` before LW's
  clock edge and applies SW using the captured address/data/enable.
- **Validation:** the initial-5 program subsequently passed with sum 15; the
  retained report `reports/day11-program2-review2/core.xml` has six passing
  cases. The saved initial-0 variant also passes in the latest 28-case regression.
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

## DAY16 — v0.3 branch-extension review findings

These were intermediate issues found and corrected during the owner-written
branch extension. They are retained as development history; the current
working files passed lint, regression, and synthesis.

- A temporary `NOEN` spelling typo was corrected to `NONE` for the default
  branch type.
- A BEQ expected tuple was temporarily duplicated while extending the decoder
  vector list. The v0.3 checkpoint had 29 total vectors; the current v0.5 list
  has one BEQ vector and 42 total vectors.
- A trailing-comma/`NULLPORT` interface issue appeared while adding the new
  decoder output. The `branch_type` port is now declared and connected through
  the decoder/core interface; current Verilator and Yosys runs report no such
  source error.
- The core branch selector was completed only after the decoder `branch_type`
  signal was wired into the core. The final path selects equality, signed
  less-than, unsigned less-than, or the inverse relation.

At the v0.3 checkpoint, reverse ordering and equality were not exercised for
BLT/BGE/BLTU/BGEU. The v1.0 directed cases now exercise those paths, and the
initial-one program case runs through the loop once. No RTL bug was found by
these added cases.

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

## DAY18 — v0.5 U/J and jump review findings

These intermediate issues were found and corrected while reviewing the
owner-written v0.5 implementation. The final source passes lint, component/core
tests, the 28-case regression, and generic synthesis.

- U/J immediate work initially contained syntax/concatenation issues. The
  corrected J immediate is sign-extended and ordered as
  `{{12{instr[31]}}, instr[19:12], instr[20], instr[30:21], 1'b0}`; component
  vectors cover positive fields, a negative `-4` value, and the invalid-type
  default.
- The decoder draft temporarily duplicated the LUI path where JALR was needed,
  selected the wrong immediate type for JALR, and accepted JALR without its
  required `funct3=000` qualification. The corrected decoder uses I immediate
  and leaves invalid JALR funct3 values at safe defaults.
- Core integration review found an unpacked `pc_plus_4` declaration, incorrect
  repeated writeback-select cases, a reversed ALU-A choice, and an incomplete
  jump combinational case. The corrected path uses packed 32-bit `pc_plus_4`,
  four distinct writeback choices, PC only when `alu_a_pc=1`, and complete
  defaults so no latch is inferred.
- The jump test initially recorded the terminal sentinel PC as an executed
  instruction. Moving the stop check before appending the PC made `visited`
  represent only fetched/executed instructions.

The final directed path is `0 -> 4 -> 8 -> 16 -> 20 -> 40 -> 44`; it checks
both link values, JALR bit-0 clearing, and that skipped writes do not occur.
Target bit 1/misalignment behavior remains an explicit coverage and feature
boundary, not a resolved bug.

## Template for future real issues

Copy this template when a new issue is actually encountered.

## YYYY-MM-DD — Short issue title

- **Date:** YYYY-MM-DD
- **Symptom:** What was observed, including the failing command or test.
- **Root cause:** Why the problem occurred.
- **Fix:** What changed to correct it.
- **Lesson learned:** What should be remembered or improved next time.
