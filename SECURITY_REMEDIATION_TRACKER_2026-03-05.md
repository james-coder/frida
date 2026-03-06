# Security Remediation Tracker (2026-03-05)

Scope: `frida` superproject + checked-out submodules.

This file tracks all identified issues, remediation plan, test coverage, and commit mapping.

## Findings

1. High: Potential overflow in Darwin injector string handling
- Risk: Unbounded `strcpy` into fixed buffers for `dylib_path`, `entrypoint_name`, `entrypoint_data`.
- Affected file: `subprojects/frida-core/src/darwin/frida-helper-backend-glue.m`
- Notes: `g_assert`-based guard is insufficient in production builds.
- Status: [x] Closed (Group A, `frida-core` commit `352f699`)

2. High: Potential overflow in FreeBSD injector trampoline data
- Risk: Unbounded `strcpy` into fixed-size fields (`fifo_path`, `so_path`, `entrypoint_name`, `entrypoint_data`, etc.).
- Affected file: `subprojects/frida-core/src/freebsd/binjector-glue.c`
- Status: [x] Closed (Group A, `frida-core` commit `352f699`)

3. High: Potential overflow in QNX injector trampoline data
- Risk: Unbounded `strcpy` into fixed-size fields (`so_path`, `entrypoint_name`, `entrypoint_data`).
- Affected file: `subprojects/frida-core/src/qnx/qinjector-glue.c`
- Status: [x] Closed (Group A, `frida-core` commit `352f699`)

4. Medium: Build-time code execution risk from `eval()` in releng config parsing
- Risk: `eval()` on config-derived strings can execute arbitrary Python when fed untrusted files.
- Affected files:
  - `releng/machine_file.py`
  - `releng/env_generic.py`
  - `releng/deps.py`
- Status: [ ] Open (Group B pending superproject pointer update)

5. Low/Medium: Build-time unsafe deserialization from `pickle.loads`
- Risk: `pickle.loads` on attacker-controlled `builddir/frida-env.dat` can execute arbitrary code.
- Affected file: `releng/meson_make.py` (producer in `releng/meson_configure.py`)
- Status: [ ] Open (Group C pending superproject pointer update)

## Remediation Groups

Group A: Injector hardening (`frida-core`)
- Files:
  - `subprojects/frida-core/src/darwin/frida-helper-backend-glue.m`
  - `subprojects/frida-core/src/freebsd/binjector-glue.c`
  - `subprojects/frida-core/src/qnx/qinjector-glue.c`
- Strategy:
  - Replace unbounded copies with checked bounded copies.
  - Convert assertions used as security guards into runtime validation with structured errors.
  - Fail injection cleanly when user-supplied strings exceed target buffer capacity.

Group B: Remove risky `eval()` from releng configuration paths
- Files:
  - `releng/machine_file.py`
  - `releng/env_generic.py`
  - `releng/deps.py`
- Strategy:
  - Replace direct `eval()` with strict parsing where possible (literal parsing / controlled expression parsing).
  - Keep required expression capability, but only through constrained evaluators and explicit allowed symbols.

Group C: Replace pickle-based env state serialization
- Files:
  - `releng/meson_configure.py`
  - `releng/meson_make.py`
- Strategy:
  - Migrate `frida-env.dat` to safe JSON schema.
  - Keep compatibility fallback only if needed and documented; otherwise fully remove `pickle.loads` path.

## Validation Plan

For each group:
- Run targeted automated checks relevant to changed code.
- Verify no regressions in affected scripts/build entrypoints.
- Run repository status check before commit.

## Commit Plan

- Commit 1: Group A (injector overflow hardening)
- Commit 2: Group B (`eval()` hardening in releng)
- Commit 3: Group C (remove pickle deserialization)

Each commit will include:
- Changed files only for its group.
- Detailed multi-line commit message with threat model and rationale.
- Push after tests pass.

## Execution Log

- Group A completed in submodule:
  - Repo: `james-coder/frida-core`
  - Commit: `352f699`
  - Checks: attempted `meson` build smoke (blocked by missing `valac`), plus targeted static validation for removed unsafe copy/assert pattern in affected files.
