# Scheme Daemon Implementation

## What

Implement the actual Redox scheme protocol handlers for `stored` and `profiled`.
The core logic exists (handles, path resolution, lazy extraction, profile mapping,
control interface) — what's missing is the `SchemeSync` trait implementation that
bridges the Redox kernel's scheme request protocol to that core logic.

## Why

The `scheme-native-package-manager` change (archived 2026-03-09) created all the
platform-independent business logic and wired the daemons into the CLI, build system,
and module system. But the `scheme.rs` files are stubs with commented pseudo-code.

Without the actual scheme implementation:
- `snix stored` and `snix profiled` exit with "not yet implemented"
- The `store:` and `profile:` schemes can't be used by other Redox programs
- Lazy extraction on first access doesn't work
- The profile union view is just a concept, not reality

## Scope

1. Add `redox_scheme`, `libredox`, and `syscall` as Cargo dependencies (cfg-gated)
2. Implement `SchemeSync` for `StoreSchemeHandler` in `stored/scheme.rs`
3. Implement `SchemeSync` for `ProfileSchemeHandler` in `profiled/scheme.rs`
4. Wire daemon main loops (Socket, register_sync_scheme, request loop)
5. Add VM tests proving the daemons work on real Redox
