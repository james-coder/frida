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
- Status: [x] Closed (Group B, `releng` commit `9ffc1cb`)

5. Low/Medium: Build-time unsafe deserialization from `pickle.loads`
- Risk: `pickle.loads` on attacker-controlled `builddir/frida-env.dat` can execute arbitrary code.
- Affected file: `releng/meson_make.py` (producer in `releng/meson_configure.py`)
- Status: [x] Closed (Group C, `releng` commit `25de252`)

6. High: Missing peer signature verification in manual pairing verification flow
- Risk: Device peer identity may not be cryptographically authenticated in `verify_manual_pairing`, enabling impersonation/MITM in this path.
- Affected file:
  - `subprojects/frida-core/src/fruity/xpc.vala`
- Status: [ ] Open (Group D)

7. High/Medium: Unsafe extraction and no integrity check for downloaded dependency bundles
- Risk: Downloaded archives are extracted using `tar.extractall()` without path-safety checks and without content integrity verification.
- Affected file:
  - `releng/deps.py`
- Status: [ ] Open (Group E)

8. Medium: Shell command construction with `shell=True` in releng bundle publishing
- Risk: `subprocess.run("cfcli purge " + public_url, shell=True)` introduces avoidable command-injection surface.
- Affected file:
  - `releng/deps.py`
- Status: [ ] Open (Group E)

9. Medium: Build-time unsafe deserialization in frida-core compat build helper
- Risk: `pickle.loads(base64.b64decode(args.state))` executes arbitrary code if `state` is attacker-controlled.
- Affected file:
  - `subprojects/frida-core/compat/build.py`
- Status: [ ] Open (Group F)

10. Medium (Correctness): frida-python Meson test path points to non-existent build directory
- Risk: test harness imports wrong module path; CI/local test signal is unreliable and currently failing.
- Affected file:
  - `subprojects/frida-python/meson.build`
- Status: [ ] Open (Group G)

11. Medium (Correctness): frida-tools Meson test path assumes nested frida-python subproject layout
- Risk: top-level superproject test execution fails with `ModuleNotFoundError` due incorrect `PYTHONPATH`.
- Affected file:
  - `subprojects/frida-tools/meson.build`
- Status: [ ] Open (Group G)

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

Group D: Pairing verification hardening (`frida-core`)
- Files:
  - `subprojects/frida-core/src/fruity/xpc.vala`
- Strategy:
  - Implement peer signature verification in `verify_manual_pairing`.
  - Fail closed and emit pairing failure signal on verification mismatch.

Group E: Dependency bundle fetch/extract hardening (`releng`)
- Files:
  - `releng/deps.py`
- Strategy:
  - Replace raw `tar.extractall()` with validated/safe extraction that rejects traversal and unsafe link targets.
  - Eliminate string-concatenated `shell=True` invocation for `cfcli purge`.

Group F: Remove pickle decode path from compat helper (`frida-core`)
- Files:
  - `subprojects/frida-core/compat/build.py`
- Strategy:
  - Replace pickle-based opaque state channel with safe structured encoding.
  - Ensure compile-step decoding only accepts constrained schema.

Group G: Restore reliable Python test harness paths (`frida-python` + `frida-tools`)
- Files:
  - `subprojects/frida-python/meson.build`
  - `subprojects/frida-tools/meson.build`
- Strategy:
  - Correct `PYTHONPATH` targets for superproject/subproject layout.
  - Re-run Meson test targets to validate import resolution.

## Validation Plan

For each group:
- Run targeted automated checks relevant to changed code.
- Verify no regressions in affected scripts/build entrypoints.
- Run repository status check before commit.

## Commit Plan

- Commit 1: Group A (injector overflow hardening)
- Commit 2: Group B (`eval()` hardening in releng)
- Commit 3: Group C (remove pickle deserialization)
- Commit 4: Group D (pairing verification hardening in `frida-core`)
- Commit 5: Group E (bundle extraction and shell invocation hardening in `releng`)
- Commit 6: Group F (remove `pickle.loads` path in `frida-core/compat`)
- Commit 7: Group G (Python test harness path fixes in `frida-python` + `frida-tools`)

Each commit will include:
- Changed files only for its group.
- Detailed multi-line commit message with threat model and rationale.
- Push after tests pass.

## Execution Log

- Group A completed in submodule:
  - Repo: `james-coder/frida-core`
  - Commit: `352f699`
  - Checks: attempted `meson` build smoke (blocked by missing `valac`), plus targeted static validation for removed unsafe copy/assert pattern in affected files.
- Group B completed in submodule:
  - Repo: `james-coder/releng`
  - Commit: `9ffc1cb`
  - Checks: `py_compile` on modified modules and evaluator smoke tests for representative `deps.toml` condition/meson expression forms.
- Group C completed in submodule:
  - Repo: `james-coder/releng`
  - Commit: `25de252`
  - Checks: `py_compile` on modified modules and JSON env-state round-trip smoke test.
