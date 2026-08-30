# Day 0 Environment Audit

Audit date: 2026-08-30 (Asia/Shanghai)

Project location: `/mnt/d/projects/rv32i-single-cycle` under WSL2. The project
and its `.venv` are stored on the D drive.

## System tools

| Check | Actual result |
| --- | --- |
| `uname -a` | `Linux Terry 6.18.33.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux` |
| `git --version` | `git version 2.43.0` |
| `make --version` | `GNU Make 4.3` |
| `verilator --version` | `Verilator 5.050 2026-07-01 rev v5.050` |
| `python3 --version` | `Python 3.12.3` |
| `pytest --version` | Not found on the system PATH; project-local copy is used |
| `cocotb-config --version` | Not found on the system PATH; project-local copy is used |
| `gtkwave --version` | `GTKWave Analyzer v3.3.116` |
| `yosys -V` | `Yosys 0.33 (git sha1 2584903a060)` |
| `riscv64-unknown-elf-gcc --version` | `riscv64-unknown-elf-gcc (13.2.0-11ubuntu1+12) 13.2.0` |
| `riscv64-unknown-elf-objdump --version` | `GNU objdump (2.42-1ubuntu1+6) 2.42` |
| `gh --version` | Not found |

Verilator reports revision `v5.050`; the installed build does not expose a
longer source commit hash in `verilator -V`. Its configured installation root
is `/usr/local/share/verilator`, with coroutine support enabled.

## Project Python virtual environment

The `.venv` was created with the system Python and dependencies were installed
from `requirements.txt`. Nothing was installed into the system Python.

| Check | Actual result |
| --- | --- |
| `python --version` | `Python 3.12.3` |
| `pip --version` | `pip 26.2.1` |
| `pytest --version` | `pytest 8.3.5` |
| `cocotb-config --version` | `2.0.1` |
| `pip show cocotb` | `cocotb 2.0.1`, installed in `.venv` |
| `pip show pytest` | `pytest 8.3.5`, installed in `.venv` |
| `python -m pip check` | `No broken requirements found.` |

## Audit assessment

The RTL verification toolchain meets the stated project targets. GitHub CLI is
missing, and no Git identity is configured, so GitHub publication and the
initial commit remain separate readiness items rather than RTL/toolchain
failures.

## Bootstrap validation

- `make env`: PASS. All required verification tools and project-local Python
  package versions were printed successfully.
- `make lint`: PASS with visible expected bootstrap warnings. Verilator reports
  `UNUSEDSIGNAL` for `a_i`, `b_i`, and `cin_i`, plus `UNDRIVEN` for `sum_o` and
  `cout_o`, because the full-adder body is intentionally TODO.
- `make test`: EXPECTED FAILURE. Verilator compilation and linking succeeded,
  cocotb started, and all eight input combinations ran. Seven combinations
  mismatched because both unimplemented outputs remain zero.
- `make waves`: EXPECTED FAILURE for the same functional reason, after creating
  a valid nonempty FST trace at `waves/full_adder.fst`.

`-Wno-fatal` is used only to keep the visible TODO warnings from stopping the
bootstrap simulation. No warning class is disabled or hidden. There are no
known RTL verification infrastructure blockers at this checkpoint.
