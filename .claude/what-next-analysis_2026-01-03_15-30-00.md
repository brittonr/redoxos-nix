# RedoxOS Nix Build: What Next Analysis

**Created**: 2026-01-03 15:30:00
**Status**: Analysis Complete

## Executive Summary

The RedoxOS Nix build system is in excellent shape. Cloud Hypervisor support was just completed and is fully working. The main priorities now are:

1. **Housekeeping**: Sync documentation with implementation status
2. **Testing**: Enable and verify host tool tests
3. **Expansion**: Add more userspace packages
4. **Exploration**: Investigate SNIX integration opportunities

## Current State Assessment

### Recently Completed Work

| Feature | Status | Commit |
|---------|--------|--------|
| Cloud Hypervisor support | FULLY WORKING | 7fd9240 |
| Reproducible builds (SOURCE_DATE_EPOCH) | Complete | e7435a4 |
| Flake modularization (flake-parts) | Complete | 904041a |
| Treefmt + git-hooks integration | Complete | 140f659 |
| Legacy build script cleanup | Complete | c32b24b |

### Build System Health

All core components build successfully:
- Host tools: cookbook, redoxfs, installer
- Cross-compilation: relibc, kernel, bootloader, base
- Userspace: ion, helix, binutils, extrautils, uutils, sodium, netutils
- Infrastructure: initfs, bootstrap, disk image
- Runners: QEMU (headless + graphical), Cloud Hypervisor (headless + network)

### Uncommitted Changes

```
Modified:
  CLAUDE.md           - Needs Cloud Hypervisor status update (says "issues" but it's working)
  flake.lock          - Input updates
  redox-src           - Local modifications for virtio fixes
  result, result-fd   - Build artifacts

Untracked:
  .claude/cloud-hypervisor-plan_2026-01-03_11-57-25.md  - Ready to commit
  snix-analysis.md    - Reference documentation
```

## Priority 1: Documentation Sync (Immediate)

### Issue: CLAUDE.md is Out of Date

The CLAUDE.md file says:
```
# WARNING: Has PCI BAR allocation issue preventing RedoxFS mount
nix run .#run-redox-cloud-hypervisor      # Boots to kernel, RedoxFS mount fails
```

But the cloud-hypervisor-plan document and implementation show this is FIXED and FULLY WORKING.

**Action**: Update CLAUDE.md to reflect working status.

### Issue: Runner Script Comment is Stale

`nix/pkgs/infrastructure/cloud-hypervisor-runners.nix` lines 64-69 still reference a "KNOWN ISSUE" that's been fixed.

**Action**: Update comments to reflect working status.

## Priority 2: Host Tool Testing

Currently disabled tests in host packages:

| Package | Tests | Reason |
|---------|-------|--------|
| cookbook | doCheck = false | Unknown |
| redoxfs | doCheck = false | Unknown |
| installer | doCheck = false | Unknown |

**Action**: Investigate why tests are disabled. If they pass, enable them for CI/CD confidence.

## Priority 3: Userspace Expansion

Current userspace packages:
- ion (shell)
- helix (editor)
- binutils (binary utilities)
- extrautils (grep, tar, gzip, less, etc.)
- uutils (coreutils replacement)
- sodium (vi-like editor)
- netutils (networking utilities)

**Potential additions** (from https://static.redox-os.org/pkg/x86_64-unknown-redox/):
- orbital (display server)
- orbterm (terminal emulator)
- orbutils (GUI utilities)
- games (tetris, snake, etc.)
- ca-certificates (TLS support)
- openssh (remote access)
- curl/wget (HTTP clients)
- git (version control)

## Priority 4: SNIX Exploration

The `snix-analysis.md` document outlines potential benefits:
- 10x faster evaluation (bytecode VM vs tree-walking)
- 90% storage reduction (content-addressed deduplication)
- Pluggable build backends (OCI, microVM, gRPC)

**Research questions**:
1. Can SNIX be used as an alternative evaluator for this project?
2. Would content-addressed storage benefit RedoxOS builds?
3. Could a microVM builder integrate with Cloud Hypervisor?

## Priority 5: CI/CD Pipeline

The project has git-hooks configured but may benefit from:
- GitHub Actions / GitLab CI for automated builds
- Cachix for binary caching
- Automated testing of QEMU and Cloud Hypervisor boot

## Recommended Next Actions

### Immediate (Today)

1. **Update CLAUDE.md** - Remove outdated Cloud Hypervisor warning
2. **Update cloud-hypervisor-runners.nix** - Remove stale "KNOWN ISSUE" comment
3. **Commit changes** - Stage all modifications with proper commit message

### Short Term (This Week)

4. **Enable host tool tests** - Investigate and re-enable doCheck for cookbook, redoxfs, installer
5. **Clean up result symlinks** - Run `nix run .#clean-results`
6. **Upstream virtio fixes** - Consider contributing pcid/virtio changes back to RedoxOS

### Medium Term (This Month)

7. **Add orbital/orbterm** - Graphical environment support
8. **Add networking packages** - openssh, curl, ca-certificates
9. **Investigate SNIX** - Prototype using snix-eval for faster iteration

### Long Term (Future)

10. **CI/CD automation** - Automated builds on GitLab/GitHub
11. **Binary cache** - Cachix or self-hosted S3 cache
12. **Additional architectures** - aarch64-unknown-redox support

## Technical Debt

| Item | Priority | Effort |
|------|----------|--------|
| Stale CLAUDE.md comments | High | Low |
| Stale runner script comments | High | Low |
| Disabled host tool tests | Medium | Medium |
| 20+ result symlinks in project root | Low | Low |
| Unused legacy build scripts | Low | Low |

## Decision Points

Before proceeding with expansion, consider:

1. **Package strategy**: Port from official RedoxOS recipes or build from source?
2. **Testing strategy**: How to test cross-compiled packages?
3. **Versioning strategy**: Pin specific RedoxOS versions or track master?
4. **Binary caching**: Self-hosted or Cachix?

## Conclusion

The project is in a mature, well-architected state. Cloud Hypervisor support marks a significant milestone - running a Rust OS in a Rust VMM is philosophically aligned. The immediate next step is documentation sync, followed by expanding the userspace package set.

The SNIX analysis suggests interesting future directions for build system innovation, but the current Nix-based approach is working well and doesn't require immediate changes.
