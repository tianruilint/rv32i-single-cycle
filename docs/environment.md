# Project Environment Audit

## Current v1.0 timing-tool addition — 2026-09-22

The authoritative checkout and WSL Ubuntu-24.04 toolchain remain D:-backed.
Yosys 0.33 mapped the core to the pinned Nangate45 typical Liberty library.
Standalone OpenSTA 2.6.0 from the OpenROAD 2024-12-14 Ubuntu 22.04 package
was extracted under ignored `build/timing/`; it was not installed system-wide.
A compatible `tcl-tclreadline` runtime was also extracted locally. The STA
command, library hash, assumptions, and observed warning are in
[`timing/README.md`](../timing/README.md); the checked report is in
`reports/timing/core_setup_nangate45_typ_10ns.txt`. The startup message
`Failed to load tclreadline.tcl` is nonfatal for this batch run. No physical
implementation or memory timing model was installed or validated.

The earlier DAY16 tool inventory and Git authentication notes below are
historical observations, not a claim about a fresh system-wide installation.

## Historical DAY16 audit — 2026-09-19

The authoritative checkout is `D:\projects\rv32i-single-cycle`, mounted at
`/mnt/d/projects/rv32i-single-cycle` in WSL distribution `Ubuntu-24.04`.
The ChatGPT project's synced mirror is a read-only reference, not the checkout.

The v0.3 regression, lint, and synthesis checks used this same verified
environment; no toolchain change was made. The previous `make env` output is
retained locally as `reports/day14-environment.log`.

| Tool/check | Observed result |
| --- | --- |
| WSL Git | 2.43.0 |
| GNU Make | 4.3 |
| Verilator | 5.050, revision v5.050 |
| System `python3 --version` | 3.12.3 |
| Project `.venv/bin/python --version` | 3.12.3 |
| Project pip | 26.2.1 |
| Project pytest | 8.3.5 |
| Project cocotb | 2.0.1 |
| GTKWave | 3.3.116 |
| Yosys | 0.33, git sha1 2584903a060 |
| RISC-V GCC | riscv64-unknown-elf GCC 13.2.0 |
| RISC-V objdump | binutils 2.42 |
| WSL GitHub CLI | 2.98.0; existing login available for repository Git operations |

The cocotb embedded-Python startup separately reports Python 3.12.4, while the
shell version commands above report 3.12.3. Both observations are preserved;
this closeout did not investigate or change interpreter/library linkage.
The latest seven-group v0.3 regression passed 24/24 cases in this environment;
the core target passed 11/11. The v0.3 generic synthesis run reported 5456
cells with 0 structural-check problems. These are not physical area or timing
results.

System PATH does not provide pytest/cocotb-config, which is expected: the
Makefile uses the project-local `.venv`. Do not reinstall working dependencies.
The RISC-V toolchain is installed, but current programs are literal instruction
words in Python; automatic assembly/image generation is not implemented.
No STA tool/library/constraint setup was validated at the DAY16 checkpoint;
the v1.0 addition above supersedes that historical limit.

### Git authentication used for closeout

Windows Git could not complete a noninteractive pull with its current credential
helper. The existing WSL GitHub CLI login worked. A one-command credential
helper override was used; no token was written into the repository and no
global Git setting was changed. Fast-forward-only pull reported
`Already up to date.` before documentation edits.

If the same local setup needs the existing WSL CLI credential helper, run from
the repository root in WSL:

```sh
git -c credential.helper= -c 'credential.helper=!gh auth git-credential' pull --ff-only
```

Do not start a new login or copy credentials unnecessarily. Commit and push
remain explicit user actions/authorizations; the current DAY16 session has
authorization for a combined commit and push, but not a tag or release.

## Historical Day 0 audit

The following is the original bootstrap record, not a description of current
RTL completeness, GitHub CLI availability, or current regression status.

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
