# P1 — RV32I Single-Cycle Processor with Automated RTL Verification

> Project master plan for Digital IC internship preparation.
>
> This file is the single source of truth for P1.
>
> Every Codex session MUST read this file before modifying the project.

## Current checkpoint and session boundary — 2026-09-22

**P1 v1.0 is the completed single-cycle engineering baseline.** The 37-type
RTL remains unchanged. v2.0 is the next milestone; no pipeline code is part of
this checkpoint. The requested Git closeout does not include a tag or release.

- Implemented: the 37-instruction single-cycle subset, PC-indexed Python
  instruction memory, external Python data memory, program-level tests,
  BNE/BLT/BGE/BLTU/BGEU, LB/LBU/LH/LHU/SB/SH, and
  LUI/AUIPC/JAL/JALR.
- Latest full regression: seven groups, 30/30 cocotb cases, 0 failed/skipped,
  exit status 0; seed forwarding and seven per-target logs verified.
- Core target: 17/17 cases passed. Immediate and decoder tests are one case
  each with 13 and 42 vectors. The testbench does not report a dynamic-
  instruction count.
- Lint: one reviewed immediate-generator `UNUSEDSIGNAL` warning, no error.
- Generic Yosys synthesis: 6642 cells, no inferred combinational latch,
  `check -assert` reports 0 problems. Nangate45-typical mapping produced 6267
  library cells and passed the structural check. Neither count is silicon area.
- Current directed cases include initial-zero/initial-one loop behavior,
  relational branch reverse/equality decisions, and a negative taken JAL.
- Basic core-only pre-layout STA: assumed 10 ns target, +5.202 ns worst setup
  slack, WNS/TNS 0, with 15 maximum-slew violations. External memories and
  wire timing are absent; no timing-closure or Fmax claim follows.
- Misaligned halfword/word accesses that would span two aligned words are
  unsupported and unverified; no misalignment or access-fault exception exists.
- JALR target bit 0 clearing is verified. The instruction model assumes
  four-byte-aligned PCs; targets with bit 1 set and instruction-address-
  misalignment exceptions are unsupported/unverified.
- DAY12 assembler-to-image automation is deferred, not completed. Automatic
  failure-wave generation was optional and is not implemented.
- The planned 2026-09-18 v2.0 target date has passed; v1.0 has now closed, but
  v2.0 remains future work and its acceptance standard is not lowered.

Authority order: latest explicit owner instruction, latest agreed milestone
scope, this plan, then older chat/history. `docs/specification.md` describes
implemented behavior; `PROGRESS.md` records evidence and the next-session start.
Historical DAY entries below remain the original learning sequence, not a
claim that each must be repeated or occupy a calendar day.

---

# 0. Project Overview

Project Name:

**RV32I Single-Cycle Processor with Automated RTL Verification**

Primary Career Targets:

- CPU Design Intern
- RTL / Digital IC Design Intern
- IP Design Intern
- SoC Design / Integration Intern

Secondary Career Targets:

- Design Verification Intern
- Formal Verification Intern

This is the first serious and auditable Digital IC project in the internship portfolio.

The goal is NOT to build the largest possible CPU.

The goal is to build a processor that the owner:

1. genuinely understands;
2. can explain module by module;
3. can debug from waveform;
4. can verify automatically;
5. can synthesize and analyze;
6. can defend during a technical interview.

---

# 1. Core Principle

The most important rule of this project:

> Never allow the GitHub project or resume description to exceed the owner's real understanding.

This project must NOT become:

- an AI-generated CPU that the owner cannot explain;
- a copied GitHub CPU with renamed modules;
- a tutorial repository presented as original work;
- an Ibex / PicoRV32 / Rocket / PULP derivative presented as self-designed;
- a collection of features added only for resume keywords.

Every important design decision must eventually be explainable without Codex.

The project follows:

Specification
→ Architecture
→ RTL
→ Verification
→ Debug
→ Regression
→ Lint
→ Synthesis
→ STA
→ Measurement
→ Documentation

A smaller complete project is preferred to a larger incomplete project.

---

# 2. Why This Project Exists

The project was selected because current internship targets include:

- CPU architecture / CPU RTL
- Digital IC design
- RTL development
- SystemVerilog / Verilog
- hardware verification
- Python automation
- synthesis
- timing analysis
- SoC integration

The owner currently needs a genuine Digital IC project that builds fundamentals quickly.

Therefore the first project starts from a simple processor rather than immediately attempting:

- full SoC;
- industrial DMA;
- UVM;
- cache;
- Linux;
- AXI;
- MMU;
- out-of-order execution.

Those may appear later in P2/P3.

---

# 3. P1 Final Scope

P1 is divided into four maturity levels.

---

## P1 v0.0 — Learning Components

Purpose:

Learn the basic RTL components required to build a CPU.

Components include:

- Full Adder
- ALU
- Register File
- Program Counter
- Multiplexers
- Immediate Generator
- Decoder fundamentals

This stage is NOT a resume project.

Status is purely educational.

---

## P1 v0.1 — Minimum Working CPU

Goal:

Build a working RV32I-subset single-cycle CPU.

Initial instruction set:

### Arithmetic

- ADD
- ADDI
- SUB

### Logic

- AND
- ANDI
- OR
- ORI

### Comparison

- SLT
- SLTI

### Memory

- LW
- SW

### Branch

- BEQ

Target:

Approximately 12 core instructions.

The CPU must be capable of executing small programs consisting of multiple instructions.

Required modules:

- Program Counter
- Instruction Memory
- Decoder / Control Unit
- Register File
- Immediate Generator
- ALU
- Data Memory interface
- Writeback logic
- Branch logic
- Top-level processor integration

For v0.1, instruction memory is satisfied by the PC-indexed cocotb/Python model;
it is not an RTL ROM. The data-memory model also remains external. This boundary
must appear in specifications, diagrams, and any project description.

Required verification:

- automated tests;
- register-state checking;
- memory-state checking;
- branch tests;
- multi-instruction programs;
- waveform debugging.

Once v0.1 works reliably, it may be described as:

**RV32I-subset Single-Cycle Processor — In Progress**

It must NOT yet be described as a complete RV32I implementation.

---

## P1 v1.0 — Formal P1 Release

This is the REQUIRED final version of P1.

The CPU remains single-cycle.

The purpose of v1.0 is NOT to make the architecture dramatically more complicated.

The purpose is to create a complete Digital IC engineering loop.

Target instruction coverage should expand toward the practical RV32I base integer instruction set.

Planned instruction groups:

### Register-register arithmetic / logical

- ADD
- SUB
- SLL
- SLT
- SLTU
- XOR
- SRL
- SRA
- OR
- AND

### Immediate arithmetic / logical

- ADDI
- SLTI
- SLTIU
- XORI
- ORI
- ANDI
- SLLI
- SRLI
- SRAI

### Loads

- LB
- LH
- LW
- LBU
- LHU

### Stores

- SB
- SH
- SW

### Branches

- BEQ
- BNE
- BLT
- BGE
- BLTU
- BGEU

### Jump / Upper immediate

- JAL
- JALR
- LUI
- AUIPC

Instructions/features not implemented must be explicitly documented.

Examples:

- FENCE
- ECALL
- EBREAK
- privileged ISA
- CSR subsystem

Do NOT silently claim support.

---

# 4. P1 v1.0 Required Engineering Closure

P1 is NOT finished merely because instructions execute correctly.

The following sections are mandatory.

---

## 4.1 Specification

The repository must contain a written processor specification.

At minimum:

- supported instructions;
- unsupported instructions;
- data width;
- register architecture;
- memory interface;
- reset behavior;
- PC behavior;
- illegal instruction behavior;
- alignment assumptions;
- architectural limitations.

Target file:

docs/specification.md

---

## 4.2 Datapath Documentation

Must include a clear single-cycle datapath.

It should show at least:

PC
→ Instruction Memory
→ Decoder
→ Register File
→ ALU
→ Data Memory
→ Writeback

And the alternate PC paths:

PC + 4

or

branch / jump target

Target files:

docs/datapath.md

and optionally:

docs/images/datapath.png

The owner must eventually be able to redraw the important datapath during an interview.

---

## 4.3 Control Table

Document how instruction decoding generates control signals.

Examples:

- RegWrite
- ALUSrc
- MemRead
- MemWrite
- MemToReg / ResultSrc
- Branch
- Jump
- ALUOp

Target:

docs/control_table.md

Avoid unnecessary control signals.

---

# 5. RTL Architecture

Recommended module structure:

rtl/
├── alu.sv
├── register_file.sv
├── pc.sv
├── immediate_generator.sv
├── decoder.sv
├── alu_decoder.sv
├── branch_unit.sv
├── rv32i_core.sv
└── optional utility modules

Memory models should normally be outside the CPU core itself.

Example:

rtl/
├── rv32i_core.sv
├── ...
sim/
├── instruction_memory.sv
└── data_memory.sv

The CPU core should not become unnecessarily tied to one simulation memory implementation.

---

# 6. RTL Coding Rules

RTL must use synthesizable SystemVerilog.

General rules:

- combinational logic uses always_comb;
- sequential state uses always_ff;
- sequential assignments normally use <=;
- combinational procedural assignments normally use =;
- provide defaults in combinational blocks;
- avoid inferred latches;
- avoid combinational loops;
- avoid magic numbers where named constants are appropriate;
- x0 must always behave as architectural zero;
- widths must be explicit where ambiguity is possible;
- signed and unsigned operations must be intentional.

Do not optimize readability away.

For a student CPU:

clarity > clever RTL.

---

# 7. Verification Strategy

Verification is a major part of P1.

P1 is explicitly:

**RV32I Single-Cycle Processor WITH Automated RTL Verification**

Therefore screenshots of GTKWave are NOT sufficient verification.

---

## 7.1 Verification Stack

Primary simulation:

- Verilator

Primary automated verification:

- cocotb
- Python

Debug:

- waveform output
- GTKWave

Later optional tools:

- Verible lint or equivalent
- SymbiYosys / formal experiments
- riscv-formal only after architecture is stable

Do not introduce complex verification frameworks prematurely.

---

# 8. ALU Verification

ALU tests must cover:

- ADD
- SUB
- AND
- OR
- XOR
- SLL
- SRL
- SRA
- SLT
- SLTU

Important corner cases:

- zero;
- all ones;
- INT_MAX;
- INT_MIN;
- signed negative values;
- signed vs unsigned comparison;
- shifts of 0;
- shifts up to 31.

Special attention:

SLT vs SLTU

and

SRL vs SRA.

---

# 9. Register File Verification

Must verify:

- read x0 returns zero;
- writes to x0 are ignored;
- normal register write;
- two simultaneous read ports;
- reset behavior if reset is implemented;
- read-after-write behavior according to defined design semantics.

Architecture:

32 registers

x0–x31

x0 = constant zero.

---

# 10. CPU Instruction Verification

Each implemented instruction should have directed tests.

Examples:

Arithmetic:

ADD
SUB
ADDI

Logical:

AND
OR
XOR

Comparison:

SLT
SLTU

Memory:

LW
SW

Branch:

BEQ

Later all instructions in v1.0.

Tests must check architectural results, not only internal signals.

Examples:

- final register values;
- final memory values;
- final PC;
- control-flow outcome.

---

# 11. Program-Level Verification

The CPU must eventually execute small programs.

Examples:

### Program A

Simple arithmetic.

### Program B

Loop.

### Program C

Array sum.

### Program D

Memory copy.

### Program E

Fibonacci or similar control-flow example.

Programs should be compiled/assembled with the RISC-V toolchain when practical.

Pipeline is NOT required for these programs.

---

# 12. Regression

The project must eventually support a command similar to:

make test

or

make regression

Expected output concept:

ALU tests              PASS
Register file tests    PASS
Arithmetic tests       PASS
Load/store tests       PASS
Branch tests           PASS
Jump tests             PASS
Program tests          PASS

TOTAL: N/N PASS

Do not fabricate N.

Only report measured numbers.

When a test fails, output should make debugging reasonably easy.

Prefer:

test name
instruction
expected value
actual value
PC
seed if randomized

---

# 13. Bug Diary

One of the important project artifacts is:

docs/bug_diary.md

For meaningful bugs, record:

- date;
- symptom;
- failing test;
- waveform observation;
- root cause;
- fix;
- lesson learned.

Examples of useful future bugs:

- incorrect sign extension;
- incorrect B-type immediate construction;
- writing x0;
- branch target wrong by 4 bytes;
- signed comparison implemented as unsigned;
- SRA implemented as logical shift;
- memory byte-enable issue;
- JALR LSB handling error.

Do NOT intentionally invent fake bugs for the resume.

Only record bugs actually encountered.

---

# 14. Lint / RTL Quality

Before P1 v1.0 is complete:

Run lint / static checks.

Check at minimum:

- inferred latch;
- width mismatch;
- unused signals;
- incomplete case;
- combinational loops;
- multiple drivers;
- suspicious signed conversions.

Warnings must not simply be hidden.

Each important warning should be:

fixed

or

documented.

---

# 15. Synthesis

P1 v1.0 must be synthesized.

Initial open-source tool:

- Yosys

Possible later school tools may be used, but the open project should remain reproducible where possible.

Record:

- tool version;
- synthesis command;
- target library if applicable;
- cell/resource usage;
- major warnings;
- design hierarchy.

Target:

reports/synthesis/

Do NOT write area numbers on the resume until they have actually been measured under documented conditions.

---

# 16. Static Timing Analysis

P1 v1.0 should include basic timing analysis.

Goal:

Understand the single-cycle critical path.

Likely candidates include paths involving:

Register File
→ ALU
→ Data Memory
→ Writeback

or another long combinational path depending on implementation.

The project should report:

- target clock;
- critical path;
- slack;
- timing assumptions;
- timing tool / library.

Target:

reports/timing/

Do not call this industrial signoff.

This is educational / project-level STA.

---

# 17. PPA Discussion

At minimum discuss:

- Performance
- approximate Area / resource usage
- architectural tradeoffs

The important lesson:

A single-cycle CPU has CPI approximately 1 for supported instructions, but the clock period must accommodate the slowest instruction path.

Therefore:

single-cycle does NOT automatically mean fast.

This observation becomes important when comparing to P1 v2.0.

---

# 18. Documentation Required for P1 v1.0

Required:

README.md

docs/
├── specification.md
├── datapath.md
├── control_table.md
├── verification_plan.md
├── bug_diary.md
└── lessons_learned.md

reports/
├── synthesis/
└── timing/

README should contain:

1. project overview;
2. architecture;
3. supported instruction list;
4. repository structure;
5. prerequisites;
6. build instructions;
7. test instructions;
8. regression results;
9. synthesis summary;
10. timing summary;
11. known limitations;
12. future work;
13. external references / licenses.

---

# 19. Repository Structure

Target long-term structure:

```text
rv32i-single-cycle/
├── README.md
├── PROJECT_PLAN.md
├── Makefile
├── requirements.txt
├── .gitignore
│
├── docs/
│   ├── specification.md
│   ├── datapath.md
│   ├── control_table.md
│   ├── verification_plan.md
│   ├── bug_diary.md
│   ├── lessons_learned.md
│   └── images/
│
├── rtl/
│   ├── alu.sv
│   ├── register_file.sv
│   ├── pc.sv
│   ├── immediate_generator.sv
│   ├── decoder.sv
│   ├── alu_decoder.sv
│   ├── branch_unit.sv
│   └── rv32i_core.sv
│
├── tb/
│   ├── test_alu.py
│   ├── test_register_file.py
│   ├── test_core.py
│   └── common/
│
├── sim/
│   ├── instruction_memory.sv
│   └── data_memory.sv
│
├── programs/
│   ├── arithmetic.S
│   ├── branch.S
│   ├── memory.S
│   ├── fibonacci.S
│   └── generated/
│
├── scripts/
│   ├── build_program.sh
│   ├── run_regression.py
│   └── report_parser.py
│
├── reports/
│   ├── synthesis/
│   └── timing/
│
└── build/
    └── generated artifacts only
```

Exact names may evolve.

Do not reorganize the entire project without a clear reason.

---

# 20. Current Development Toolchain

Expected environment:

Windows host
+
WSL2 Ubuntu

Primary tools:

- SystemVerilog
- Verilator
- cocotb
- Python
- GTKWave
- Yosys
- RISC-V GCC toolchain
- Git
- Make

Before installing anything, Codex MUST first check whether the required tool already exists.

Do NOT:

- reinstall working packages unnecessarily;
- change tool versions without justification;
- replace the existing build flow simply because another tool is preferred.

The repository should remain reproducible.

---

# 21. Day-by-Day Development Plan

The exact number of days may change.

Understanding and correctness take priority over calendar speed.

---

## DAY00 — Environment

Goal:

Make the complete basic RTL development loop work.

Pipeline:

SystemVerilog
→ Verilator
→ cocotb
→ waveform
→ GTKWave

Also verify:

- Python
- Make
- Git
- Yosys
- RISC-V GCC

Deliverable:

A minimal RTL module can compile, simulate, automatically test and generate waveform.

---

## DAY01 — RTL Fundamentals + Full Adder

Goal:

Understand the development workflow before CPU implementation.

Topics:

- module;
- input/output;
- wire/logic;
- combinational logic;
- basic SystemVerilog syntax;
- testbench;
- cocotb;
- waveform;
- Git workflow.

Implementation:

full_adder.sv

Verification:

test_full_adder.py

Important:

Full Adder is a learning exercise.

It is NOT a resume project.

---

## DAY02 — 32-bit ALU

Goal:

Build the first real CPU datapath component.

Implement:

- ADD
- SUB
- AND
- OR
- XOR
- SLL
- SRL
- SRA
- SLT
- SLTU

Learn:

- always_comb;
- case;
- blocking assignment;
- combinational logic;
- signed vs unsigned;
- arithmetic vs logical shifts;
- parameter / enum if appropriate.

Verification:

- deterministic tests;
- corner cases;
- randomized arithmetic cases where useful.

Deliverables:

rtl/alu.sv

tb/test_alu.py

docs/alu.md if needed.

---

## DAY03 — Register File

Goal:

Implement the RISC-V architectural register file.

Architecture:

32 × 32-bit registers.

Ports:

- rs1 read
- rs2 read
- rd write

Critical rule:

x0 is always zero.

Learn:

- always_ff;
- sequential logic;
- posedge clock;
- non-blocking assignment;
- register arrays;
- synchronous writes;
- combinational reads.

Deliverables:

rtl/register_file.sv

tb/test_register_file.py

---

## DAY04 — Program Counter and Fetch

Goal:

Understand how a processor moves from instruction to instruction.

Implement:

- PC register;
- PC + 4;
- reset PC;
- next-PC selection skeleton;
- simple instruction memory model.

Learn:

- why RV32I instructions advance by 4 bytes;
- state;
- instruction address;
- fetch.

Deliverables:

rtl/pc.sv

simulation instruction memory

tests for PC behavior.

---

## DAY05 — Immediate Generator

Goal:

Decode immediate values from RISC-V instructions.

Implement immediate formats needed for current instruction subset.

Initially:

- I-type
- S-type
- B-type

Later:

- U-type
- J-type

Critical concepts:

- sign extension;
- bit extraction;
- immediate reconstruction.

Verification must strongly test negative immediates.

Deliverables:

rtl/immediate_generator.sv

tb/test_immediate_generator.py

---

## DAY06 — Instruction Decoder / Control Unit

Goal:

Convert instruction fields into datapath control decisions.

Learn:

- opcode;
- funct3;
- funct7;
- rd;
- rs1;
- rs2.

Implement control for the initial instruction subset.

Output signals may include:

- RegWrite
- ALUSrc
- MemWrite
- ResultSrc
- Branch
- ALU operation

Exact architecture may be refined.

Deliverables:

rtl/decoder.sv

rtl/alu_decoder.sv if separated

decoder tests.

---

## DAY07 — Datapath Integration Part 1

Goal:

Connect:

PC
Instruction
Register File
Immediate Generator
ALU
Decoder

Initial target:

execute register-register and immediate arithmetic instructions.

Examples:

ADDI
ADD
SUB
AND
OR
SLT

At this stage no memory instruction is required yet.

---

## DAY08 — Load / Store

Goal:

Add data-memory behavior.

Support:

LW
SW

Learn:

- effective address;
- memory read;
- memory write;
- writeback selection.

Tests should verify actual memory content.

---

## DAY09 — Branch

Goal:

Support conditional control flow.

Initial:

BEQ

Learn:

- comparison;
- branch immediate;
- branch target;
- next PC;
- taken vs not taken.

Test both:

taken

and

not taken.

---

## DAY10 — P1 v0.1 Integration

Goal:

Reach the first working processor milestone.

Initial supported instructions approximately:

ADD
ADDI
SUB
AND
ANDI
OR
ORI
SLT
SLTI
LW
SW
BEQ

Run automated regression.

No manual-only verification.

P1 v0.1 is achieved only when all required tests pass.

---

## DAY11 — Program Execution

Run multi-instruction programs.

Examples:

- arithmetic chain;
- loop;
- array sum;
- simple memory copy.

Verify architectural state at program completion.

---

## DAY12 — Regression Infrastructure

Closeout scope: one-command execution, XML case statistics, failure-target
reporting, per-target logs, and cocotb seed forwarding are implemented.
Automatic assembly/program-image build remains deferred; current programs are
machine-word dictionaries. Wave generation remains an explicit user command.
See `docs/verification_plan.md` for runner limitations and actual results.

Improve:

- automatic program build;
- automatic test execution;
- clear PASS/FAIL output;
- deterministic reproduction;
- logs;
- waveform generation on failure if useful.

Target:

one-command regression.

---

## DAY13 — Lint + Synthesis Introduction

Run:

- lint;
- RTL static checks;
- Yosys synthesis.

Analyze warnings.

Do not suppress errors blindly.

---

## DAY14 — P1 v0.1 Release

2026-09-15 decision: close v0.1 here, update the repository documentation, and
commit/push the owner-written DAY11–12 work together with the documentation.
Do not create a Git tag or GitHub Release without a separate request. A dirty
worktree before the authorized closeout commit is not a completed Git handoff.
Future implementation belongs to a new conversation.

Deliver:

- working CPU;
- tests;
- regression;
- initial documentation;
- architecture diagram;
- clean repository;
- Git tag / release if appropriate.

At this point P1 becomes a legitimate project artifact.

---

# 22. Post-DAY18 — P1 v1.0 Development

The v1.0 single-cycle implementation is the stable baseline for subsequent
v2.0 pipeline work. Keep it available and verified while pipeline stages are
added incrementally.

The following v0.2-v0.5 names label the Phase A-D work. They are intermediate
checkpoints on the path to v1.0, not release tags.

| Milestone | Scope | Main work/evidence |
| --- | --- | --- |
| v0.2 / Phase A | XOR/XORI, shifts, SLTU/SLTIU | Decoder extension, shift-encoding qualification, instruction tests; **closed 2026-09-16** |
| v0.3 / Phase B | BNE, BLT, BGE, BLTU, BGEU | Branch selection, signed/unsigned comparison, taken/not-taken tests; **implementation checkpoint 2026-09-19; reverse/equality cases added in v1.0** |
| v0.4 / Phase C | LB/LBU/LH/LHU/SB/SH | Byte-addressed memory contract, write masks, load selection/extension, tests; **closed 2026-09-20, misaligned halfword/word accesses remain unsupported/unverified** |
| v0.5 / Phase D | LUI, AUIPC, JAL, JALR | U/J immediates, PC and writeback choices, jump tests; **closed 2026-09-21, instruction-target alignment exceptions remain unsupported/unverified** |
| v1.0 | Stable expanded single-cycle CPU | **Closed 2026-09-22:** 30/30 regression, lint review, generic and library-mapped synthesis, basic core-only STA with limits recorded |
| v2.0 | Five-stage pipeline, after the v1.0 stability gate | Pipeline registers, forwarding, hazards, stall/bubble/flush, tests |

### Next-session starting task: v2.0 pipeline contract

1. Preserve the v1.0 source and regression as the reference baseline.
2. Explain RAW hazards, forwarding limits, load-use stalls, stall/bubble/flush,
   wrong-path side effects, and CPI versus instruction latency/throughput.
3. Define IF/ID/EX/MEM/WB responsibilities and pipeline-register contents
   before creating pipeline RTL. Start with the smallest hazard-free sequence.
4. Keep the v1.0 external-memory and timing limits explicit; no physical Fmax
   or area claim follows from the educational core-only STA.

The owner knows C and is learning Python as a verification tool. Group related
instructions to keep progress fast. Do not repeat completed DAY11 exercises,
turn routine report parsing into a separate course, or add unrequested tests.
Keep core RTL, important reference expectations, and primary tests owner-written.
Auxiliary changes may be assisted within explicit authorization.

For every new instruction:

1. update specification;
2. implement RTL;
3. write tests;
4. run full regression;
5. commit separately where practical.

Do NOT add many instructions in one giant unreviewable commit.

---

# 23. P1 v1.0 Completion Criteria

P1 v1.0 is DONE only when:

## Architecture

- [x] supported instruction set documented
- [x] datapath documented
- [x] control logic documented
- [x] limitations documented

## RTL

- [x] CPU core synthesizes
- [x] no known unintended latch
- [x] no unresolved multiple-driver issue
- [x] x0 behavior correct
- [x] reset behavior defined
- [x] signed operations correct

## Verification

- [x] instruction-level tests exist
- [x] program-level tests exist
- [x] register checks automatic
- [x] memory checks automatic
- [x] branches tested both directions
- [x] regression runs with one command
- [x] regression passes

## Engineering

- [x] lint reviewed
- [x] synthesis completed
- [x] synthesis report stored
- [x] STA completed
- [x] timing report stored
- [x] major warnings reviewed

## Documentation

- [x] README complete
- [x] specification complete
- [x] datapath diagram complete
- [x] control table complete
- [x] verification plan complete
- [x] bug diary contains actual development bugs
- [x] measured results documented
- [x] third-party references acknowledged

P1 v1.0 meets these educational engineering criteria within the external-memory
and core-only timing assumptions documented above. These checks do not assert
formal equivalence, complete ISA coverage, timing closure, or physical PPA.

---

# 24. P1 v2.0 — Five-stage CPU Design Extension

The owner has chosen to pursue v2.0 after the v1.0 closeout. It remains a
separate milestone, not an accomplished feature of the current repository.

It is intended especially for stronger CPU Design applications.

Architecture:

Classic five-stage pipeline:

IF
→ ID
→ EX
→ MEM
→ WB

Planned features:

- pipeline registers;
- forwarding;
- hazard detection;
- load-use stall;
- branch flush;
- bubbles;
- performance counters;
- CPI measurement.

The project should then compare:

Single-cycle CPU

vs

Five-stage pipeline CPU

Possible measurements:

- Fmax
- area
- CPI
- execution cycles
- control complexity

Important:

P1 v2.0 MUST NOT start until P1 v1.0 architecture is understood and stable.

Pipeline code must not be generated wholesale by Codex before the owner understands:

- RAW hazard;
- forwarding;
- load-use hazard;
- stall;
- bubble;
- flush.

---

# 25. Things Explicitly OUT OF SCOPE for P1 v1.0

Unless the plan is intentionally updated later:

- cache;
- MMU;
- Linux boot;
- virtual memory;
- branch prediction;
- superscalar execution;
- out-of-order execution;
- multicore;
- full privileged architecture;
- sophisticated interrupt system;
- industrial AXI subsystem;
- UVM verification environment;
- physical design signoff;
- tapeout.

These are NOT required to make P1 strong.

Do not add resume-keyword features merely because they sound impressive.

---

# 26. Role of Codex

Codex is:

- tutor;
- code reviewer;
- verification assistant;
- environment assistant;
- debugging assistant;
- automation assistant.

Codex is NOT the CPU designer.

---

# 27. Codex Allowed Tasks

Codex may:

- inspect repository status;
- explain existing RTL;
- generate module skeletons;
- create TODOs;
- generate testbench infrastructure;
- extend cocotb tests;
- diagnose compilation failures;
- inspect logs;
- help read waveforms;
- review RTL for latches / width / signedness issues;
- create Makefile targets;
- build regression scripts;
- run lint;
- run synthesis;
- parse reports;
- improve documentation;
- propose design alternatives;
- ask the owner to choose between architectural options.

These capabilities are not blanket write permission. In mentor mode, provide
specifications, TODOs, and staged review; the owner writes core RTL and primary
test logic. Direct RTL/test replacement requires an explicit request. Document
updates, Git commits, pushes, tags, and releases each require applicable owner
authorization. Do not treat a request to run tests as permission to change them.

For the 2026-09-22 v1.0 closeout, the owner explicitly authorized document
updates and a Git push. A commit is the necessary intermediate step; no tag,
release, or pipeline RTL change is part of this authorization.

---

# 28. Codex Forbidden Behaviors

Codex must NOT:

1. generate the complete CPU in one shot;
2. replace multiple unfinished student modules with a finished external implementation;
3. import an existing RISC-V core;
4. copy Ibex / PicoRV32 / Rocket / VexRiscv / PULP code;
5. silently expand project scope;
6. claim unsupported instructions;
7. fabricate regression results;
8. fabricate timing / area numbers;
9. fabricate bugs;
10. rewrite large sections without explaining why;
11. hide warnings merely to make CI green;
12. add advanced features before the current milestone is complete;
13. change toolchains unnecessarily;
14. make resume claims on behalf of the owner that are not supported by the repository.

---

# 29. Required Codex Workflow for Every Session

At the beginning of EVERY Codex work session:

## Step 1 — Read project authority files

Read:

- `PROJECT_PLAN.md`;
- `README.md`;
- the relevant specifications, tests, and RTL for the current task.

---

## Step 2 — Inspect repository

Check:

git status

recent commits

current files

existing tests

current build status

Do not assume previous session state.

---

## Step 3 — Identify current milestone

State explicitly:

Current Day:

Current module:

Current milestone:

Current failing tests:

Next smallest deliverable:

Do not jump forward.

---

## Step 4 — Explain before major implementation

Before adding a new architectural module, briefly explain:

- what the module does;
- why the CPU needs it;
- inputs;
- outputs;
- combinational or sequential;
- how it will be tested.

The owner is learning CPU architecture through this project.

---

## Step 5 — Implement only today's scope

Prefer:

one small correct milestone

over

many incomplete features.

---

## Step 6 — Test

Run the relevant tests.

If relevant, run regression.

Never report success without actually executing the tests.

---

## Step 7 — Summarize

At the end report:

1. what changed;
2. files changed;
3. tests executed;
4. test results;
5. bugs found;
6. unresolved issues;
7. what the owner should understand;
8. recommended next task.

---

# 30. Learning Rule

Whenever Codex writes or modifies important RTL, the owner must be able to eventually answer:

- What does this module do?
- Why does it exist?
- Is it combinational or sequential?
- Why?
- What are its inputs?
- What are its outputs?
- What happens on reset?
- What corner cases exist?
- How is it tested?
- What bug would occur if this logic were wrong?

For CPU-level logic additionally:

- Which instructions use this path?
- Which control signals select it?
- What happens to PC?
- What happens to rd?
- Does memory participate?
- How does signedness matter?

---

# 31. Git Strategy

Use incremental commits.

Good examples:

feat(alu): implement RV32I ALU operations

test(alu): add signed and shift corner cases

feat(regfile): implement 32x32 register file

fix(branch): correct B-immediate sign extension

test(core): add branch-taken regression

docs: document single-cycle datapath

Avoid one massive commit containing the entire CPU.

Git history is part of the project's auditability.

---

# 32. Resume Rules

Resume claims must come AFTER implementation.

---

## Before v0.1

Do not list P1 as a completed project.

---

## After v0.1

Possible:

RV32I Single-Cycle Processor — In Progress

Possible bullet:

Implemented a 32-bit single-cycle RISC-V processor in SystemVerilog supporting arithmetic, logical, load/store, and branch operations, with automated Verilator/cocotb verification.

Only use this if true.

Instruction count may be included only if measured and accurate.

---

## After v1.0

Possible project title:

RV32I Single-Cycle Processor with Automated RTL Verification

Potential bullet structure:

Designed and implemented a synthesizable 32-bit RV32I single-cycle processor in SystemVerilog, integrating instruction decode, register file, ALU, branch/jump control, load/store and writeback datapaths.

Second bullet may discuss:

automated regression

lint

synthesis

STA

measured timing

but ONLY after results exist.

Example structure:

Built an automated cocotb/Verilator regression environment covering instruction- and program-level behavior, and analyzed synthesis and timing results under documented constraints.

Do not invent numeric results.

---

# 33. Interview Readiness Definition

The project is not resume-ready merely because GitHub is green.

The owner should eventually be able to explain:

### CPU fundamentals

- ISA vs microarchitecture
- PC
- register file
- ALU
- instruction decoding
- immediate generation
- memory access
- writeback
- branch
- jump

### RISC-V

- instruction fields
- R/I/S/B/U/J formats
- x0
- sign extension
- branch immediates
- load/store behavior

### RTL

- combinational vs sequential logic
- always_comb
- always_ff
- blocking vs non-blocking
- latch
- reset
- signed / unsigned

### Verification

- directed testing
- random testing
- reference model
- regression
- waveform debugging

### Implementation

- synthesis
- STA
- critical path
- clock period
- area/performance tradeoff

If the owner cannot explain a resume bullet, the bullet must be simplified.

---

# 34. Relationship to P2

P2 will NOT replace P1.

P2 will build a small SoC around a processor.

Possible direction:

**RV32I-based Small SoC with Memory-Mapped Peripherals**

Potential components:

- CPU
- memory map
- bus
- UART
- GPIO
- timer
- interrupt
- bare-metal C software

Possible bus:

- simple custom bus first;
- APB or AXI4-Lite later.

P2 targets:

- SoC Integration
- RTL Design
- IP Design
- Embedded Hardware

---

# 35. Relationship to P3

P3 will deepen either:

RTL/IP Design

or

Design Verification.

Possible direction:

**Parameterized DMA / Data Mover with Verification**

Potential features:

- AXI4-Lite control
- AXI / streaming data path
- FIFO
- backpressure
- interrupt
- multiple transfer sizes
- arbitration
- error handling

Verification extension:

- cocotb
- assertions
- formal
- coverage
- optionally UVM later

P3 targets:

- IP Design
- SoC
- RTL
- DV
- Formal

---

# 36. Overall Portfolio Story

The final portfolio should ideally tell one coherent story:

P1
Build a processor.

↓

P2
Build a system around the processor.

↓

P3
Add a real data-movement IP and deeper verification.

This is preferred over three unrelated tutorial projects.

Conceptually:

CPU
+
Bus
+
Memory
+
Peripherals
+
DMA
+
Verification

This progressively builds toward practical SoC design.

---

# 37. Priority Order

When time is limited:

Priority 1:

P1 working RTL

Priority 2:

P1 automated verification

Priority 3:

P1 architectural understanding

Priority 4:

P1 documentation

Priority 5:

P1 synthesis / STA

Priority 6:

additional instructions

Priority 7:

pipeline

Priority 8:

advanced features

A working, understood CPU beats an incomplete advanced CPU.

---

# 38. Project Anti-Goals

The following outcomes are considered failures even if the code runs:

### Failure A

Codex writes the entire CPU and the owner cannot explain it.

### Failure B

A public CPU is copied and renamed.

### Failure C

The repository claims "RV32I" but supports only a handful of instructions without documenting that limitation.

### Failure D

Only screenshots are provided as verification.

### Failure E

No automatic regression exists.

### Failure F

Timing / area numbers appear without reproducible conditions.

### Failure G

README claims UVM/formal/signoff without those flows actually existing.

### Failure H

Complexity is increased solely for resume keywords.

---

# 39. Current Golden Rule

Before starting a new feature ask:

> Does this feature increase actual interview-ready RTL/CPU ability, or does it merely make the project title look bigger?

If the answer is only "it looks bigger":

Do not add it yet.

---

# 40. End Goal

When P1 is finished, the owner should be capable of taking a blank sheet and explaining:

1. how an RV32I instruction enters the processor;
2. how it is decoded;
3. how operands are obtained;
4. how the ALU executes it;
5. how load/store interacts with memory;
6. how results are written back;
7. how branches and jumps change PC;
8. how RTL implements that behavior;
9. how the implementation was verified;
10. how failures were debugged;
11. how the RTL was synthesized;
12. what determines the single-cycle critical path;
13. what would have to change to pipeline the CPU.

At that point P1 has achieved its real purpose.

It is no longer merely a student CPU.

It is evidence that the owner understands the complete basic RTL design workflow.

---

# 41. Milestone Sequence

Development should continue sequentially.

Do NOT start P2 or P3 before P1 reaches a stable milestone.

Planned milestone sequence:

DAY01
Full Adder / basic RTL workflow

↓

DAY02
32-bit ALU

↓

DAY03
Register File

↓

DAY04
PC / Instruction Fetch

↓

DAY05
Immediate Generator

↓

DAY06
Decoder / Control

↓

DAY07–10
Single-Cycle CPU integration

↓

P1 v0.1

↓

Expanded instructions
Verification
Lint
Synthesis
STA
Documentation

↓

P1 v1.0

↓

Optional five-stage pipeline

↓

P1 v2.0

---

# 42. Instruction to Future Codex Sessions

If you are Codex reading this file:

DO NOT immediately begin coding.

First:

1. inspect the repository;
2. identify the current milestone;
3. run the existing tests;
4. read the relevant RTL;
5. state what is already complete;
6. state what remains;
7. propose the smallest next implementation step.

The owner is using this project both to build a portfolio and to learn Digital IC design from first principles.

Therefore:

**Teaching clarity, correctness, reproducibility and auditability are part of the project requirements.**

Do not optimize only for speed.
