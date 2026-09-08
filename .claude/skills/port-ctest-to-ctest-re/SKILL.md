---
name: ctest-re-add
description: Ports existing CTest invocations to ctest-re while preserving test selection, configuration, parallelism, environment, working directory, dashboards, and failure reporting. Use after adding cmake-re or when asked to run CMake tests through ctest-re locally or in CI.
---

# Port CTest Invocations to ctest-re

Use the `ctest-re` binary shipped with the same CMake RE release used to configure and build the project. Never assume compatibility from old documentation: inspect the installed binary first.

## Discover the Installed Interface

Run:

```bash
cmake-re --version
ctest-re --version
ctest-re --help
```

Record the tested version. The `cmake-re` and `ctest-re` versions must match.

As of `v0.0.87`, `ctest-re` accepts:

```text
ctest-re [--distributed] [<ctest-args>]
```

Normal CTest arguments are forwarded. `--distributed` belongs to `ctest-re`; all remaining arguments retain their CTest meaning.

## Find Every Invocation

Search workflows, scripts, Makefiles, presets, and documentation:

```bash
rg -n --hidden \
  --glob '!build/**' --glob '!_build/**' --glob '!.git/**' \
  '(^|[;&|()[:space:]])ctest([[:space:]]|$)|cmake[[:space:]]+--build.*(--target|-t)[=[:space:]]*(test|check)|cmake[[:space:]]+--build.*--run-test'
```

Also inspect:

- `testPresets` in `CMakePresets.json`
- `ctest_test(...)` dashboard scripts
- shell functions or variables that resolve to `ctest`
- Docker entrypoints and composite GitHub Actions

Classify each use:

1. Direct test execution: `ctest ...`
2. Preset execution: `ctest --preset ...`
3. Dashboard/script mode: `ctest -S ...` or `ctest -D ...`
4. Indirect test target: `cmake --build ... --target test`
5. Configure/build/test orchestration: `ctest --build-and-test ...`

## Port Conservatively

### Direct and preset execution

Replace only the executable:

```diff
-ctest -C Release --output-on-failure -j 4
+ctest-re -C Release --output-on-failure -j 4
```

Preserve:

- working directory
- argument order and quoting
- `-C/--build-config`
- `-R`, `-E`, labels, fixtures, resource specs, repeat policy, stop time, timeout, and parallelism
- environment variables such as `CTEST_OUTPUT_ON_FAILURE`
- output capture and exit-code handling

When the build was configured by `cmake-re`, generated tests can be wrapped
with `tipi-test-driver`. A raw `ctest` call may then fail unless that driver is
on `PATH`; `ctest-re` resolves the matching driver itself. Treat this as an
expected wrapper difference, not evidence that the compiled test is broken.

### Dashboard and script mode

Port only after testing that the installed `ctest-re` forwards the exact mode correctly. If unsupported, keep the original `ctest` invocation and report it rather than changing semantics.

### Indirect test targets

Do not mechanically replace `cmake --build --target test` with `ctest-re`. First determine:

- the build directory
- configuration for a multi-config generator
- environment and dependencies added by the build target
- whether the target is really CTest's generated `test` target

Use `ctest-re --test-dir <build-dir>` plus equivalent CTest arguments only when parity is demonstrated.

### `ctest --build-and-test`

Do not reduce this to a test-only command. Keep configure and build under `cmake-re`, then run `ctest-re` in the resulting build tree:

```bash
cmake-re -S <source> -B <build> <configure-options>
cmake-re --build <build> <build-options>
ctest-re --test-dir <build> <test-options>
```

## Local Versus Distributed Execution

Default to local `ctest-re`; this preserves behavior while using the CMake RE test wrapper.

Add `--distributed` only when the user explicitly requests remote test execution and the environment provides the required RBE configuration. For `v0.0.87`, these may include:

- `RBE_service`
- `RBE_server_address`
- `RBE_exec_root`
- `RBE_platform`
- `RBE_tls_client_auth_key`
- `RBE_tls_client_auth_cert`

Never put RBE credentials or certificate contents in the repository. Reference GitHub secrets or environment-specific paths.

## GitHub Actions

Install CMake RE once, then verify both binaries:

```yaml
- name: Verify CMake RE tools
  run: |
    cmake-re --version
    ctest-re --version
```

Port the test step without disturbing its working directory or environment:

```yaml
- name: Test
  working-directory: build/cmake-re
  env:
    CTEST_OUTPUT_ON_FAILURE: "1"
  run: ctest-re -C Release
```

Pin the CMake RE version when CI reproducibility matters. If using the official installer, set `TIPI_INSTALL_VERSION` to the tested release rather than silently tracking `master`.

## Validate

1. Run the original CTest command and record discovered, passed, skipped, and failed tests.
2. Run the ported `ctest-re` command against a clean build produced by the matching `cmake-re`.
3. Compare test counts, labels/filters, configuration, working directory, output-on-failure behavior, and exit status.
4. Test zero-match and intentionally failing filters if the workflow depends on their exit behavior.
5. Validate workflow YAML and shell syntax.

If a full containerized run is unavailable, validate `ctest-re --help`, preserve the invocation mechanically, and say that execution parity remains unproven.

## Report

State:

- matching `cmake-re` and `ctest-re` versions tested
- every invocation changed
- invocations intentionally left unchanged and why
- test-count and exit-status parity
- whether execution was local or distributed

Do not claim the test port is complete if dashboard, preset, indirect, or generated invocations were skipped.