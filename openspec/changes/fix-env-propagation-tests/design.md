## Context

60/62 self-hosting tests pass. The two failures (`env-propagation-simple`, `env-propagation-heavy`) test whether `option_env!("LD_LIBRARY_PATH")` returns `Some` at rustc compile time. This var is set by cargo via `Command::env()` but is NOT covered by the `--env-set` workaround (which only handles `CARGO_PKG_*`, `OUT_DIR`, and `cargo:rustc-env` values).

Three patches already exist in the build:
1. **patch-relibc-execvpe.py** — adds `execvpe()` to relibc (PATH search with explicit envp)
2. **patch-rustc-execvpe.py** — patches Rust std's `do_exec()` to call `execvpe()` on Redox, bypassing the global `environ` pointer
3. **patch-relibc-dso-environ.py** — makes `getenv()` self-initialize from `__relibc_init_environ` when the DSO's local `environ` is null

Despite all three, `option_env!("LD_LIBRARY_PATH")` returns `None` in rustc. The var survives exec (kernel puts it in the new process's envp), but rustc's DSO-linked `librustc_driver.so` can't read it through `getenv()` or `std::env::var()`.

Basic env propagation (bash→bash, ion→ion) passes in functional tests. The failure is specific to the compile-time env lookup path through DSO-linked rustc.

## Goals / Non-Goals

**Goals:**
- Diagnose exactly where LD_LIBRARY_PATH disappears in the cargo→rustc exec→DSO chain
- Fix environ propagation so `option_env!()` sees process env vars in DSO-linked binaries
- Get `env-propagation-simple` and `env-propagation-heavy` to PASS (62/62)
- Evaluate whether `--env-set` scope can be reduced

**Non-Goals:**
- Fixing `env-propagation-simple` and `env-propagation-heavy` by adding LD_LIBRARY_PATH to `--env-set` (that's a workaround, not a fix)
- Rewriting the entire DSO initialization architecture
- Fixing the `parallel-jobs2` cc-wrapper linker crash (separate bug)

## Decisions

### 1. Diagnostic-first approach

Add a diagnostic patch that traces the exact state at each stage:
- After execve: does the kernel's envp contain LD_LIBRARY_PATH?
- In relibc_start: does `environ` get set correctly?
- In run_init: does `__relibc_init_environ` get injected into the DSO?
- In getenv: does the fallback trigger? What does it find?
- In rustc's option_env expansion: what does `std::env::var()` return?

This avoids guessing and identifies the exact broken link.

**Rationale**: Three patches already target this chain and all three "should" work. Blind fixes risk adding a fourth patch that also "should" work. Trace first.

### 2. Likely root cause: Rust std `environ` access bypasses getenv

Rust std's `std::sys::env::vars()` reads the `environ` global pointer directly via `extern { static environ: ... }`. In a DSO, this resolves to the DSO's own `environ` copy (via BSS), which is null if init_array didn't set it. The `getenv()` fallback patch only helps code that calls `getenv()` — but Rust std may iterate `environ` directly for `std::env::var()`.

If confirmed, the fix is to ensure the DSO's `environ` pointer is set during init_array (not just as a getenv fallback). The init_array currently fails because it reads `__relibc_init_environ` via GLOB_DAT (main binary's copy, which is null at init_array time). Possible fixes:
- **Option A**: Set DSO `environ` lazily in Rust std's env module (like the getenv fallback)
- **Option B**: Defer DSO init_array environ setup to after main binary init
- **Option C**: Have `relibc_start` (in main binary) broadcast environ to all loaded DSOs after setting it

### 3. Fix in relibc, not in rustc

The fix belongs in relibc's environ initialization path, not in rustc. Any DSO-linked Rust binary should have working environ, not just rustc.

### 4. Preserve --env-set as defense-in-depth

Even after fixing process environ, keep `--env-set` for `CARGO_PKG_*` vars. Belt and suspenders — if process environ breaks again, cargo builds still work.

## Risks / Trade-offs

- **[Risk] Diagnosis takes longer than expected** → Cap diagnostic phase at one VM boot cycle. If the trace clearly shows the break point, fix it. If unclear, try Option C (broadcast) as a brute-force fix.
- **[Risk] Fix breaks other DSO-linked binaries** → Run full self-hosting test suite (62 tests) after any relibc change.
- **[Risk] Removing --env-set too aggressively** → Keep --env-set, only shrink scope after 62/62 confirmed.
- **[Trade-off] Option C (broadcast) is ugly but reliable** → Acceptable as a first fix; can be refined later with proper init ordering.
