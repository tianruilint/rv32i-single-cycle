# v0.5 Learning and Ownership Notes

Recorded at DAY18, 2026-09-21. These are topics practiced or explained, not a
claim that every topic has passed an independent oral assessment.

## Owner-written work

- Component and integrated CPU RTL, reviewed incrementally.
- Primary cocotb stimulus, expected results, and architectural assertions.
- v0.3 decoder branch-type selection, core branch comparison logic, and
  primary branch cases.
- v0.4 decoder extension, byte/halfword lane selection, write strobes, and
  primary subword memory cases.
- v0.5 U/J immediate extension, upper-immediate and jump decoder controls,
  PC/ALU/writeback selection, and primary LUI/AUIPC/JAL/JALR cases.
- PC-indexed program execution and external Python memory behavior.
- The regression runner, rebuilt incrementally after the owner rejected an
  earlier complete assistant-written version.

Assistant support included specifications/hints, code review, running checks,
waveform interpretation, authorized build configuration, and documentation.
This does not transfer authorship of a complete externally supplied CPU to
the owner; no existing third-party CPU implementation was imported.

## Hardware and verification lessons

- PC is DUT state. A program test reads PC to choose an instruction; it does
  not compute the program in Python and assign the final result to the DUT.
- Instructions encode operands and operations, not the contents of external
  data memory. LW therefore needs `data_read_data` from the environment; SW
  exports the value to be stored through `data_write_data`.
- Register-file reads are combinational; enabled writes occur at a rising edge.
  x0 behavior and uninitialized x1-x31 must be treated separately.
- Convert simulator values to Python integers before dictionary addressing.
  Supply load data before the consuming edge and sample settled signals.
- A byte address and an aligned 32-bit external word are different interface
  concepts. Low address bits select little-endian byte lanes; the Python model
  owns word assembly and preserves bytes whose store strobe is clear.
- `data_write_en` identifies the decoded store path, while
  `data_write_strb[3:0]` identifies the byte lanes that may change. Store data
  must be placed in the same lanes as the strobe bits.
- LB/LH sign-extend and LBU/LHU zero-extend. Byte operations may use any byte
  address; halfword and word alignment limits must be stated separately from
  the absence of a misalignment trap.
- BEQ targets the current PC plus its signed immediate. The core compares
  operands directly and suppresses register/memory write side effects.
- `branch_type` separates BEQ/BNE from signed BLT/BGE and unsigned BLTU/BGEU.
  `$signed` is required for the signed relation; the unsigned relation uses the
  original 32-bit vectors. Greater-or-equal branches use the inverse of the
  corresponding less-than result.
- Passing one ordering for each relational pair does not prove the reverse
  ordering or equality boundary. Those remain explicit v0.3 follow-up checks.
- A successful simulation tests sampled behaviors; lint and synthesis examine
  different structural properties. None alone proves complete correctness.
- A register array need not map to SRAM. The observed generic flow used
  flip-flops and read-selection MUXes; cell counts are not physical area.
- Test-case counts, vector counts, instruction support, and coverage are
  different quantities. Historical evidence must not be relabeled as current
  default regression coverage.
- LUI writes the U immediate itself, whereas AUIPC adds that immediate to the
  current instruction PC. This requires a separate ALU-A choice rather than
  changing the meaning of the existing rs1 path.
- JAL and JALR both write `PC+4`, but their targets come from different paths:
  `PC+Jimm` versus `rs1+Iimm`. JALR then clears target bit 0.
- A jump test must prove both where execution goes and what does not execute.
  Checking the skipped destination registers catches wrong-path side effects.
- Clearing JALR bit 0 is not an instruction-address-misalignment implementation.
  Under a four-byte-aligned instruction interface, target bit 1 still needs an
  explicit contract or exception path.

## v0.3 through v0.5 instruction-group lessons

- Signed `SLT/SLTI` and unsigned `SLTU/SLTIU` use different comparison
  interpretations even though both return only 0 or 1. `SLTIU` still receives
  a sign-extended immediate before the unsigned comparison.
- Register shifts use only the low five bits of the shift source in RV32. A
  register value of 32 therefore behaves as a shift amount of 0.
- `SRL/SRLI` fill with zero, while `SRA/SRAI` preserve the sign bit for a
  signed right shift.
- Ordinary I-type upper bits are immediate data. SLLI/SRLI/SRAI are the special
  I-type forms whose upper field must be qualified as `0000000` or `0100000`.
- `reg_write` controls the register-file write port. `data_write_en` is the
  external data-memory write signal used by SW; an ALU instruction should not
  assert the latter.
- The v0.5 immediate and decoder tests are one cocotb case each containing 13
  and 42 vectors; the core target reaches 15 cases and the full regression
  reaches 28 cases. The implemented subset contains 37 instruction types.
  Dynamic instruction
  executions are not instrumented. Instruction-type, vector, cocotb-case, and
  dynamic-instruction counts remain separate evidence categories.
- A cocotb case must initialize its own clock, reset, inputs, and architectural
  preconditions. The v0.4 lane-boundary case initially failed this rule and
  produced a simulator shutdown followed by misleading zero-time failures.

## Explicit remaining understanding checks

The owner recognized state storage and the 32x32 bit count. The explanation
that read MUXes select a register's data based on its address was provided after
the owner's initial wording referred only to selecting 0/1. Do not infer a
complete independent datapath explanation from that exchange alone.

Before claiming synthesis/STA expertise, independently explain what a generic
cell count means, what a warning does and does not establish, and why a testbench
clock period does not prove an achievable hardware frequency. STA has not run.
Before calling jump handling independently mastered, explain why JAL uses the
current PC, why link data is `PC+4`, why JALR clears bit 0, and why skipped
instructions cannot have side effects.

## Next-session teaching contract

Keep owner effort on architectural decisions, core RTL, reference expectations,
and important test logic. Explain Python through the owner's C background when
needed. Auxiliary argument parsing, log formatting, and Git commands are not
separate line-by-line assignments. Group related instructions, preserve actual
verification, honor explicitly skipped cases, and start at the nearest
unfinished feature. v1.0 begins with a stabilization audit of the implemented
37-instruction core, its remaining high-value verification gaps, and the tool
requirements for honest basic timing analysis; do not begin the v2.0 pipeline
from this closeout note alone.
