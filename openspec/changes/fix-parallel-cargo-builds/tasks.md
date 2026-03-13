## 1. Build and run self-hosting-test

- [ ] 1.1 Build `nix build .#self-hosting-test` — captures full disk image with self-hosting profile
- [ ] 1.2 Run `nix run .#self-hosting-test` and capture serial log
- [ ] 1.3 Check `parallel-jobs2` result in serial log — if PASS, skip section 2 entirely

## 2. Diagnose and fix parallel-jobs2 crash (only if 1.3 shows FAIL)

- [ ] 2.1 Examine cc wrapper debug files in serial log (`/tmp/.cc-wrapper-raw-args`, `/tmp/.cc-wrapper-stderr`, `/tmp/.cc-wrapper-last-err`) for clues
- [ ] 2.2 Check if crash is in lld-wrapper (thread creation failure), bash cc wrapper (fd/pipe issue), or cargo's fork/exec path
- [ ] 2.3 Write and deploy the fix (modify cc wrapper, lld-wrapper, or self-hosting-test as needed)
- [ ] 2.4 Rebuild and rerun self-hosting-test to verify parallel-jobs2 PASS

## 3. Archive investigation changes

- [ ] 3.1 Archive `cargo-parallel-hang-investigation` — check off tasks 5.1-5.4 and 6.1-6.7 with notes (done/superseded/deferred), sync delta specs, move to `openspec/changes/archive/2026-03-13-cargo-parallel-hang-investigation`
- [ ] 3.2 Archive `fix-remaining-os-bugs` — check off tasks 3.1-3.5 and 4.1-4.2 with deferred notes, sync delta specs, move to `openspec/changes/archive/2026-03-13-fix-remaining-os-bugs`

## 4. Update documentation

- [ ] 4.1 Correct napkin entry for "Self-hosting test parallel-jobs2 linker crash" — update with actual root cause or move to stale claims if already fixed
- [ ] 4.2 Correct napkin claim that cc wrapper runs "lld inside clang" — the cc wrapper calls lld-wrapper directly, not clang for linking
- [ ] 4.3 Verify AGENTS.md parallel build section is accurate (fork-lock, lld-wrapper, CLONE_LOCK documentation)
- [ ] 4.4 Commit all changes
