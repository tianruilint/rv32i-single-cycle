# RV32I Single-Cycle Processor

This repository is a work-in-progress educational project for building an
RV32I single-cycle processor and verifying its RTL with automated tests.

The integrated processor does **not** exist yet. The current checkpoint has
completed and independently tested components through DAY06: a one-bit full
adder, a 32-bit ALU, a 32 x 32-bit register file, a program counter, an I/S/B
immediate generator, and a decoder for the current instruction subset.
Instruction memory, data memory, top-level integration, and RISC-V instruction
execution remain future work.

## Environment

The project targets WSL2 Ubuntu, SystemVerilog, Verilator 5.036 or newer,
cocotb 2.0.1, pytest, GTKWave, Yosys, and the
`riscv64-unknown-elf` GCC/binutils toolchain. Python verification packages are
installed in the project-local `.venv`.

From the project root:

```sh
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements.txt
make env
make lint
make test
```

Current component verification targets are:

```sh
make lint
make test
make lint-alu
make test-alu
make test-register-file
make test-pc
make test-immediate-generator
make test-decoder
```

These targets verify individual learning components; they do not run an
integrated CPU or instruction-level regression.

Generated simulator output, reports, caches, and waveforms are ignored by Git.
