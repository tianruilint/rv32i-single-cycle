# v0.2 Learning and Ownership Notes

Recorded at DAY15, 2026-09-16. These are topics practiced or explained, not a
claim that every topic has passed an independent oral assessment.

## Owner-written work

- Component and integrated CPU RTL, reviewed incrementally.
- Primary cocotb stimulus, expected results, and architectural assertions.
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
- BEQ targets the current PC plus its signed immediate. The core compares
  operands directly and suppresses register/memory write side effects.
- A successful simulation tests sampled behaviors; lint and synthesis examine
  different structural properties. None alone proves complete correctness.
- A register array need not map to SRAM. The observed generic flow used
  flip-flops and read-selection MUXes; cell counts are not physical area.
- Test-case counts, vector counts, instruction support, and coverage are
  different quantities. Historical evidence must not be relabeled as current
  default regression coverage.

## v0.2 instruction-group lessons

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
- The new owner-written decoder test remains one cocotb case containing 22
  vectors; instruction count, vector count, and cocotb case count remain
  separate evidence categories.

## Explicit remaining understanding checks

The owner recognized state storage and the 32x32 bit count. The explanation
that read MUXes select a register's data based on its address was provided after
the owner's initial wording referred only to selecting 0/1. Do not infer a
complete independent datapath explanation from that exchange alone.

Before claiming synthesis/STA expertise, independently explain what a generic
cell count means, what a warning does and does not establish, and why a testbench
clock period does not prove an achievable hardware frequency. STA has not run.

## Next-session teaching contract

Keep owner effort on architectural decisions, core RTL, reference expectations,
and important test logic. Explain Python through the owner's C background when
needed. Auxiliary argument parsing, log formatting, and Git commands are not
separate line-by-line assignments. Group related instructions, preserve actual
verification, honor explicitly skipped cases, and start at the nearest unfinished
feature. v0.3 branch implementation begins in the next conversation, not this
closeout.
