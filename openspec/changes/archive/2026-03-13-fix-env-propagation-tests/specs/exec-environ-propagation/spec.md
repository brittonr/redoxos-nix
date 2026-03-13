## ADDED Requirements

### Requirement: DSO-linked binaries see process environment variables

When a process is started via `execvpe()` with an explicit envp array, all dynamically-linked libraries loaded by `ld_so` SHALL have access to the full set of environment variables through `getenv()` and through direct `environ` pointer reads.

This applies to any DSO that statically links relibc (each DSO gets its own `environ` pointer). The DSO's `environ` SHALL be initialized to point at the process's actual environment before any application code runs.

#### Scenario: Simple cargo build with option_env

- **WHEN** cargo invokes rustc with `LD_LIBRARY_PATH` set via `Command::env()`
- **AND** rustc is dynamically linked against `librustc_driver.so`
- **THEN** `option_env!("LD_LIBRARY_PATH")` evaluated inside librustc_driver.so SHALL return `Some(...)` with the value cargo set

#### Scenario: Heavy-fork build script followed by option_env

- **WHEN** a crate's `build.rs` fork+exec's external commands (e.g., clang) 20 times
- **AND** the crate's lib target uses `option_env!("LD_LIBRARY_PATH")`
- **THEN** the env var SHALL still be visible to rustc's `option_env!()` expansion after the build script completes

#### Scenario: Rust std env::var in DSO code

- **WHEN** code inside a `.so` library calls `std::env::var("KEY")`
- **AND** `KEY` was set in the process envp at exec time
- **THEN** the call SHALL return `Ok(value)` matching the envp entry

### Requirement: environ pointer consistent across DSO and main binary

The `environ` pointer in each loaded DSO SHALL point to the same environment data as the main binary's `environ` pointer. Reads through either copy SHALL return identical results.

#### Scenario: getenv returns same value from DSO and main binary

- **WHEN** a process is started with `FOO=bar` in envp
- **AND** the main binary calls `getenv("FOO")` returning `"bar"`
- **THEN** a DSO loaded in the same process calling `getenv("FOO")` SHALL also return `"bar"`

#### Scenario: environ not null in DSO after process start

- **WHEN** ld_so finishes loading all DSOs and runs init_arrays
- **AND** `relibc_start` in the main binary sets `environ` from the kernel envp
- **THEN** the DSO's `environ` pointer SHALL be non-null before any application-level `getenv()` call

### Requirement: Self-hosting test validation

The self-hosting test suite SHALL validate environ propagation end-to-end.

#### Scenario: env-propagation-simple passes

- **WHEN** the self-hosting VM test runs the `env-propagation-simple` test
- **THEN** the test SHALL report `FUNC_TEST:env-propagation-simple:PASS`

#### Scenario: env-propagation-heavy passes

- **WHEN** the self-hosting VM test runs the `env-propagation-heavy` test
- **THEN** the test SHALL report `FUNC_TEST:env-propagation-heavy:PASS`
