## Why

Two self-hosting tests fail: `env-propagation-simple` and `env-propagation-heavy`. Both check whether `option_env!("LD_LIBRARY_PATH")` returns `Some` at compile time inside rustc. The DSO environ fix (getenv self-init from `__relibc_init_environ`) solved `env!("CARGO_PKG_NAME")` and similar vars that travel via `--env-set`, but `LD_LIBRARY_PATH` is set via `Command::env()` / process environ only — it never goes through `--env-set`. The var is lost somewhere in the `cargo → rustc` exec chain. Fixing these gets us to 62/62 self-hosting tests.

## What Changes

- Diagnose exactly where `LD_LIBRARY_PATH` disappears in the exec chain (cargo sets it → rustc should see it)
- Fix the environ propagation path so vars set via `Command::env()` survive exec() into DSO-linked binaries on Redox
- Update the two tests from expected-fail to expected-pass once the fix lands
- Remove or reduce the `--env-set` workaround scope if process environ now works end-to-end

## Capabilities

### New Capabilities

- `exec-environ-propagation`: Reliable environment variable propagation through exec() into dynamically-linked binaries on Redox. Covers the cargo→rustc path where `Command::env("KEY", "val")` must be visible to the child process's `getenv()` and rustc's `option_env!()`.

### Modified Capabilities

(none — no existing spec-level requirements change)

## Impact

- **relibc**: Likely needs changes in exec/spawn path (`src/platform/redox/`) to preserve environ across exec into DSO-linked binaries
- **ld_so**: May need changes to `run_init()` environ injection ordering
- **self-hosting-test.nix**: Two test expectations flip from FAIL to PASS
- **patch-cargo-env-set.py**: Scope may shrink (process environ handles more vars)
- **patch-relibc-dso-environ.py**: May need extension or companion patch
