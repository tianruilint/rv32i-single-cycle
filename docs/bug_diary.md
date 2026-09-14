# Bug Diary

This diary records actual development failures, not hypothetical bugs or
invented fault-injection evidence. Updated for DAY14 on 2026-09-15.

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
  cases. The saved initial-0 variant also passes in the latest 19-case regression.
- **Waveform boundary:** the retained, inspected waveform belongs to the later
  initial-zero run; no failing-waveform observation is asserted for this error.
- **Lesson:** a DUT value handle/LogicArray is not a Python integer, and a value
  stored in a Python local variable does not drive a DUT input. Type conversion
  and pre-edge data timing are both part of a correct external memory model.

Earlier register-enable, writeback-direction, LW/SW, and BEQ integration issues
are recorded in the DAY10 section of `PROGRESS.md`. Their history is preserved
there without manufacturing missing failure logs or waveform measurements.

## Template for future real issues

Copy this template when a new issue is actually encountered.

## YYYY-MM-DD — Short issue title

- **Date:** YYYY-MM-DD
- **Symptom:** What was observed, including the failing command or test.
- **Root cause:** Why the problem occurred.
- **Fix:** What changed to correct it.
- **Lesson learned:** What should be remembered or improved next time.
