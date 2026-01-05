# What Next? - Priority Analysis

Created: 2026-01-04 05:45:00 UTC

## Current System Status

The RedoxOS Nix build system is **mature and production-ready**:
- Boots successfully to interactive login prompt
- Networking functional (DHCP + static fallback)
- User authentication working (getty, login, passwd)
- Full tooling available (ion shell, helix, sodium, uutils, extrautils, netutils)
- QEMU and Cloud Hypervisor runners operational
- Graphics drivers included but no desktop environment

## Recent Accomplishments (Last 24-48 Hours)

1. **vesad fix (04:36:00)** - Resolved graphics daemon circular dependency
2. **redoxfs-ar --uid/--gid** - Fixed file ownership in Nix sandbox
3. **passwd/group/shadow format** - Corrected for Redox OS
4. **Network daemon logging** - Output redirected to log files
5. **Interactive shell login** - Now fully functional

## Priority Action Items

### TIER 1: High Value, Actionable Now

#### 1. Add CI/CD Pipeline
**Effort**: 2-3 hours
**Value**: Prevents regressions, automates quality gates

Create `.github/workflows/ci.yml`:
```yaml
name: CI
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: cachix/install-nix-action@v22
      - run: nix flake check
  build-host-tools:
    runs-on: ubuntu-latest
    steps:
      - uses: cachix/install-nix-action@v22
      - run: nix build .#fstools
  build-image:
    runs-on: ubuntu-latest
    steps:
      - uses: cachix/install-nix-action@v22
      - run: nix build .#diskImage
```

#### 2. Set Up Binary Caching (Cachix)
**Effort**: 30 minutes
**Value**: Dramatically reduces build times for contributors

Add to `flake.nix`:
```nix
nixConfig = {
  extra-substituters = [ "https://redox-nix.cachix.org" ];
  extra-trusted-public-keys = [ "redox-nix.cachix.org-1:..." ];
};
```

#### 3. Fix Hardcoded Values
**Effort**: 1 hour
**Value**: Configuration flexibility, easier updates

Files to update:
- `nix/flake-modules/overlays.nix:101` - Reference config for nightly date
- `nix/flake-modules/overlays.nix:84` - Reference config for target triple

### TIER 2: Medium Effort, High Value

#### 4. Complete Orbital Desktop (BLOCKED)
**Effort**: 1-3 days (investigation + implementation)
**Value**: Full graphical desktop environment

Current blockers (from orbital.nix):
1. redox-scheme version conflict (0.8.3 required, 0.8.2 available)
2. Nested path dependencies from base (graphics-ipc, inputd, daemon)
3. Git dependencies needing path conversion (redox-ioctl from relibc)

Approaches:
- **Option A**: Create unified workspace with all path deps vendored
- **Option B**: Pin redox-scheme 0.8.3, patch all consumers
- **Option C**: Wait for upstream crates.io publication

#### 5. Refactor Vendor Merge Duplication
**Effort**: 2-3 hours
**Value**: Better Nix cache utilization, reduced maintenance

8 files duplicate vendor merge logic. Should use `vendor.mkMergedVendor` consistently:
- relibc.nix, kernel.nix, bootloader.nix, base.nix
- mk-userspace.nix, extrautils.nix, sodium.nix, bootstrap.nix

### TIER 3: Lower Priority

#### 6. Add Package Metadata
**Effort**: 30 minutes
**Value**: Better Nix tooling integration

Add `meta.platforms` to:
- kernel.nix, bootloader.nix, base.nix

#### 7. Consolidate Shell Scripts
**Effort**: 2 hours
**Value**: Code maintenance, fewer bugs

Extract common OVMF/QEMU discovery functions to shared library.

#### 8. Fix base-src Local Path
**Effort**: 30 minutes
**Value**: Reproducibility across machines

Change from `git+file://` to proper remote URL or documented submodule.

## Recommended Next Action

**Start with CI/CD** - It's:
- High value (prevents regressions)
- Low effort (2-3 hours)
- Unblocks other work (can test changes automatically)
- No dependencies on upstream (unlike Orbital)

After CI/CD is set up, the Orbital desktop blocker becomes the primary focus for anyone wanting graphical output.

## Architecture Overview

```
flake.nix (325 lines)
├── nix/flake-modules/ (12 modules)
│   ├── packages.nix - Package exports
│   ├── config.nix - Central configuration
│   ├── toolchain.nix - Rust toolchain
│   ├── sources.nix - Patched source handling
│   ├── checks.nix - Nix checks
│   ├── apps.nix - Runnable applications
│   └── devshells.nix - Development environments
├── nix/lib/ (6 utilities)
│   ├── vendor.nix - Cargo vendor merging
│   ├── rust-flags.nix - RUSTFLAGS construction
│   ├── stub-libs.nix - Unwinding stubs
│   ├── sysroot.nix - Rust std vendoring
│   └── cross-compile.nix - Cross-compilation helpers
├── nix/pkgs/
│   ├── host/ - Native tools (cookbook, redoxfs, installer)
│   ├── system/ - Cross-compiled core (relibc, kernel, bootloader, base)
│   ├── userspace/ - Applications (ion, helix, sodium, uutils, netutils, etc.)
│   └── infrastructure/ - Disk images, VM runners
└── nix/patches/ - Source patches
```

## Package Build Status

| Category | Package | Status |
|----------|---------|--------|
| Host | cookbook | Working |
| Host | redoxfs | Working |
| Host | installer | Working |
| System | relibc | Working |
| System | kernel | Working |
| System | bootloader | Working |
| System | base | Working |
| Userspace | ion | Working |
| Userspace | helix | Working |
| Userspace | sodium | Working |
| Userspace | uutils | Working |
| Userspace | extrautils | Working |
| Userspace | netutils | Working |
| Userspace | userutils | Working |
| Userspace | binutils | Working |
| Graphics | orbdata | Working |
| Graphics | orbital | **BLOCKED** |
| Graphics | orbterm | **BLOCKED** |
| Infrastructure | diskImage | Working |
| Infrastructure | QEMU runners | Working |
| Infrastructure | Cloud Hypervisor | Working |

## Test Commands

```bash
# Build and run (headless)
nix run .#runQemu

# Build and run (graphical - shows driver init but no desktop)
nix run .#runQemuGraphical

# Build disk image only
nix build .#diskImage

# Run all checks
nix flake check

# Cloud Hypervisor with networking
sudo nix run .#setupCloudHypervisorNetwork  # Once
nix run .#runCloudHypervisorNet
```
