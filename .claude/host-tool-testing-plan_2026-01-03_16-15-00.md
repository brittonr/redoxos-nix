# Host Tool Testing Plan: cookbook, redoxfs, installer

**Created**: 2026-01-03 16:15:00
**Status**: Planning Complete

## Executive Summary

This plan outlines how to enable tests for the three host tools (cookbook, redoxfs, installer) that currently have `doCheck = false`. Each package has different test requirements and complexity levels.

## Package Analysis

### 1. redoxfs (HIGH VALUE - Enable First)

**Location**: `nix/pkgs/host/redoxfs.nix`
**Build System**: Crane (craneLib.buildPackage)
**Current State**: `doCheck = false`

#### Test Infrastructure Available

| Type | Count | Location |
|------|-------|----------|
| Unit tests | 20+ | `src/*.rs` (allocator, block, dir, header, htree, tree) |
| Integration tests | 10+ | `tests/tests.rs` |
| Fuzz tests | 3 | `fuzz/fuzz_targets/` |

#### Test Dependencies
```toml
[dev-dependencies]
assert_cmd = "2.0.17"
```

#### Key Findings
- Tests create temporary disk images and mount/unmount them
- Integration tests use `assert_cmd` to test the `redoxfs` binary
- Tests require FUSE for mounting (already in buildInputs)
- Some tests call `sync` and `fusermount` system commands

#### Sandbox Concerns
- **FUSE mounting**: Tests mount filesystems using FUSE, which requires `/dev/fuse`
- **Temporary files**: Tests create `image*.bin` files in current directory
- **System commands**: Uses `sync`, `fusermount3 -u` for unmounting

#### Recommended Approach
```nix
# Option A: Enable unit tests only (safe for sandbox)
doCheck = true;
cargoTestExtraArgs = "--lib";  # Skip integration tests requiring FUSE

# Option B: Full tests with sandbox relaxation
doCheck = true;
__noChroot = true;  # Required for FUSE access
```

#### Risk Assessment
- **Unit tests**: LOW RISK - Pure Rust, no I/O
- **Integration tests**: MEDIUM RISK - Require FUSE, may need sandbox relaxation

---

### 2. installer (MEDIUM VALUE)

**Location**: `nix/pkgs/host/installer.nix`
**Build System**: stdenv.mkDerivation (manual cargo)
**Current State**: `doCheck = false`

#### Test Infrastructure Available

| Type | Count | Location |
|------|-------|----------|
| Unit tests | 0 | None found in src/lib.rs |
| Integration tests | 0 | No tests/ directory |

#### Key Findings
- **No tests exist in the upstream source**
- The installer is a relatively simple tool that copies files to RedoxFS images
- Uses patched `ring` crate (crates.io version instead of git) for pregenerated assembly

#### Recommended Approach
```nix
# Simply enable check phase - it will be a no-op if no tests exist
doCheck = true;

checkPhase = ''
  runHook preCheck
  cargo test --release
  runHook postCheck
'';
```

#### Risk Assessment
- **LOW RISK**: No tests to run, enabling doCheck is harmless
- Future-proofs the build for when tests are added upstream

---

### 3. cookbook (LOW VALUE - Deprioritize)

**Location**: `nix/pkgs/host/cookbook.nix`
**Build System**: Crane (craneLib.buildPackage)
**Current State**: `doCheck = false`
**Source**: `inputs.redox-src` (main Redox repository, not a dedicated repo)

#### Test Infrastructure Available

The cookbook source is part of the main Redox repository. Investigation needed to determine test availability.

#### Key Findings
- Source comes from `redox-src` input (main repository)
- Cookbook is the package manager/build tool (`repo` binary)
- Likely has network-dependent functionality that may not test well in sandbox

#### Recommended Approach
```nix
# Conservative: Enable with library tests only
doCheck = true;
cargoTestExtraArgs = "--lib";
```

#### Risk Assessment
- **MEDIUM RISK**: Unclear what tests exist
- Package management tools often have network-dependent tests

---

## Implementation Plan

### Phase 1: redoxfs Unit Tests (Immediate - Low Risk)

**Goal**: Enable unit tests for redoxfs without sandbox changes

**Changes to `nix/pkgs/host/redoxfs.nix`**:
```nix
craneLib.buildPackage {
  pname = "redoxfs";
  version = "unstable";

  inherit src;

  cargoExtraArgs = "--locked";

  # Enable unit tests only (skip integration tests requiring FUSE)
  doCheck = true;
  cargoTestExtraArgs = "--lib";

  nativeBuildInputs = with pkgs; [
    pkg-config
  ];

  buildInputs = with pkgs; [
    fuse
    fuse3
  ];

  meta = { ... };
}
```

**Validation**:
```bash
nix build .#redoxfs
# Verify tests ran in build log
```

### Phase 2: installer Check Phase (Low Risk)

**Goal**: Add check phase even though no tests exist currently

**Changes to `nix/pkgs/host/installer.nix`**:
```nix
# After buildPhase, add:
checkPhase = ''
  runHook preCheck
  cargo test --release || echo "No tests found"
  runHook postCheck
'';

doCheck = true;
```

### Phase 3: cookbook Exploration (Research Required)

**Goal**: Understand what tests exist and enable if safe

**Steps**:
1. Clone `redox-src` and examine Cargo.toml for cookbook
2. Run `cargo test --lib --no-run` to see what tests compile
3. Enable conservatively with `--lib` flag

### Phase 4: Full Integration Tests (Future - Requires Sandbox Changes)

**Goal**: Enable full integration tests for redoxfs

**Approach**:
- Create a separate `redoxfs-test` check that runs with `__noChroot = true`
- Add to `nix/flake-modules/checks.nix`:

```nix
# Integration test with relaxed sandbox
redoxfs-integration-test = pkgs.runCommand "redoxfs-integration-test" {
  __noChroot = true;
  buildInputs = [ packages.redoxfs pkgs.fuse3 ];
} ''
  # Set up FUSE
  mkdir -p /tmp/test-mount

  # Run integration tests
  cd ${inputs.redoxfs-src}
  cargo test --test tests

  touch $out
'';
```

---

## Risk Matrix

| Package | Test Type | Risk | Sandbox | Priority |
|---------|-----------|------|---------|----------|
| redoxfs | Unit | Low | Standard | HIGH |
| redoxfs | Integration | Medium | Relaxed | LOW |
| installer | All | Low | Standard | MEDIUM |
| cookbook | Unit | Medium | Standard | LOW |

---

## Rollback Strategy

If tests fail or cause issues:

1. **Immediate**: Revert `doCheck = false` for affected package
2. **Short-term**: Add `cargoTestExtraArgs = "--skip problematic_test"`
3. **Long-term**: Fix tests upstream or disable specific tests

---

## Monitoring

Track test results via:
1. `nix build .#redoxfs 2>&1 | grep -E "(test|PASSED|FAILED)"`
2. Add test checks to `nix/flake-modules/checks.nix`
3. CI/CD integration (future)

---

## Implementation Order

```
Step 1: [SAFE] Enable redoxfs unit tests (--lib only)
   |
   v
Step 2: [SAFE] Add installer checkPhase
   |
   v
Step 3: [TEST] Verify builds still work
   |
   v
Step 4: [COMMIT] Commit changes
   |
   v
Step 5: [FUTURE] Investigate cookbook tests
   |
   v
Step 6: [FUTURE] Enable integration tests with sandbox relaxation
```

---

## Code Changes Summary

### File: `nix/pkgs/host/redoxfs.nix`
```diff
-  doCheck = false;
+  doCheck = true;
+  cargoTestExtraArgs = "--lib";
```

### File: `nix/pkgs/host/installer.nix`
```diff
+  checkPhase = ''
+    runHook preCheck
+    cargo test --release || true
+    runHook postCheck
+  '';
+
-  doCheck = false;
+  doCheck = true;
```

### File: `nix/pkgs/host/cookbook.nix`
```diff
-  doCheck = false;
+  doCheck = true;
+  cargoTestExtraArgs = "--lib";
```

---

## Dependencies

- No new Nix dependencies required
- `assert_cmd` is a dev-dependency pulled automatically
- FUSE already in buildInputs for redoxfs

---

## Success Criteria

1. `nix build .#redoxfs` completes with test output visible
2. `nix build .#installer` completes (may show "no tests")
3. `nix build .#cookbook` completes with test output visible
4. `nix flake check` passes with new test checks
5. No sandbox violations or build failures

---

## References

- [Crane API Documentation](https://github.com/ipetkov/crane/blob/master/docs/API.md)
- [craneLib.cargoTest](https://crane.dev/API.html#cranelibcargotest) - For separate test derivations
- [Nix sandbox documentation](https://nixos.org/manual/nix/stable/command-ref/conf-file.html#conf-sandbox)
- redoxfs source: `gitlab.redox-os.org/redox-os/redoxfs`
