# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a RedoxOS build environment using Nix flakes. The project contains:
- A complete Nix-based build system for RedoxOS (replacing Make/Podman workflows)
- Various build scripts for relibc and cross-compilation toolchain setup
- Integration with Nixtamal for input pinning

## Architecture

### Directory Structure
- `/` - Root contains Nix flake configuration and build scripts
- `redox-src/` - RedoxOS source tree (when cloned)
- `nix/tamal/` - Nixtamal configuration for input management
- `nix/pkgs/` - Additional Nix packages (if present)

### Key Components (from flake.nix)

The Nix flake builds RedoxOS components in dependency order:
1. **Host Tools**: cookbook, redoxfs, installer - tools that run on the build machine
2. **Cross-compilation Toolchain**: relibc (C library), sysroot vendoring
3. **System Components**: kernel, bootloader, base (essential system binaries)
4. **Boot Infrastructure**: bootstrap loader, initfs (initial RAM filesystem)
5. **Disk Image**: Complete bootable UEFI disk image

### Cross-compilation Target
- Primary target: `x86_64-unknown-redox`
- Uses LLVM/Clang toolchain with custom linker scripts
- Requires `-Z build-std` for building Rust standard library

## Common Commands

### Building with Nix

```bash
# Build host tools (works with standard sandbox)
nix build .#cookbook    # Package manager
nix build .#redoxfs     # Filesystem tools
nix build .#fstools     # All host tools combined

# Cross-compiled components (all work with standard sandbox)
nix build .#relibc
nix build .#kernel
nix build .#bootloader
nix build .#base
nix build .#initfs

# Complete disk image
nix build .#diskImage

# Run in QEMU (works with standard sandbox)
nix run .#run-redox            # Headless mode with serial console
nix run .#run-redox-graphical  # Graphical mode with display window
```

### Development Shells

```bash
# Enter development shell
nix develop          # Default pure Nix environment
nix develop .#native # Full native environment with all tools
nix develop .#minimal # Minimal environment

# Legacy nix-shell
nix-shell           # Default shell
nix-shell -A native # Native shell
```

### Build Scripts (Legacy/Debug)

```bash
./build-env.sh       # Set up build environment without patchelf
./build-relibc.sh    # Build relibc manually
./build-simple.sh    # Simple build attempt
./build-with-wrappers.sh # Build with compiler wrappers
```

### Running Tests

Currently, tests are disabled in most packages due to cross-compilation. To run tests when available:
```bash
cargo test --target x86_64-unknown-redox
```

## Important Technical Details

### Vendor Management
The build uses offline Cargo vendoring with version-aware merging:
- Project dependencies are vendored first
- Sysroot dependencies (for `-Z build-std`) are merged with version conflict resolution
- Checksums are regenerated after vendor merging

### Linker Configuration
- Uses `ld.lld` (LLVM linker) for all Redox target builds
- Custom linker scripts for bootstrap loader
- Stub libraries for unwinding functions (since `panic=abort` is used)

### Known Issues and Workarounds

1. **Vendor Checksum Issues**: Git dependencies sometimes have incorrect checksums after vendoring. The build regenerates all checksums using Python scripts.

2. **Duplicate Symbols**: Uses `--allow-multiple-definition` linker flag to resolve conflicts between relibc's bundled core/alloc and `-Z build-std` versions

3. **UEFI AES Intrinsics Bug**: Bootloader forces software AES implementation with `--cfg aes_force_soft` to avoid LLVM codegen issues

## Nixtamal Integration

The project includes Nixtamal configuration in `nix/tamal/manifest.kdl` for input pinning. This manages nixpkgs versions and other dependencies with KDL-based configuration.

## Environment Variables

Key environment variables set by the build:
- `CARGO_BUILD_TARGET`: x86_64-unknown-redox
- `RUST_SRC_PATH`: Path to Rust standard library source
- `NIX_SHELL_BUILD=1`: Indicates Nix shell environment
- `PODMAN_BUILD=0`: Disables Podman-based builds

## Running and Testing

### Successful Boot Output

When running with `nix run .#run-redox` or after building the disk image, you should see:
1. UEFI bootloader starting
2. RedoxFS detection and mounting
3. Kernel initialization with memory detection
4. Driver initialization
5. "Redox OS boot complete!" message

### Expected Errors in Headless Mode

The following errors are normal when running in headless/serial console mode:
- **ps2d panic**: "No such device" - PS/2 driver fails because there's no keyboard/mouse in headless QEMU
- **PCIe info warning**: Falls back to PCI 3.0 configuration space
- These errors don't prevent the system from booting successfully

### Building and Running the Disk Image

```bash
# Build the complete disk image
nix build .#diskImage

# Run the built image in QEMU
nix run .#run-redox            # Headless mode (auto-selects display after 3 seconds)
nix run .#run-redox-graphical  # Graphical mode with interactive display selection

# Or run manually with the built image
# Headless mode:
qemu-system-x86_64 \
  -m 2048 \
  -enable-kvm \
  -bios /path/to/OVMF.fd \
  -drive file=result/redox.img,format=raw \
  -nographic

# Graphical mode:
qemu-system-x86_64 \
  -m 2048 \
  -enable-kvm \
  -bios /path/to/OVMF.fd \
  -drive file=result/redox.img,format=raw \
  -vga std \
  -display gtk
```

### Running Modes

**Headless Mode** (`nix run .#run-redox`):
- Uses serial console output
- Automatically selects display resolution after 3 seconds
- Good for CI/testing or when you don't need graphics
- Exit with Ctrl+A then X

**Graphical Mode** (`nix run .#run-redox-graphical`):
- Opens a QEMU window with full graphics
- Interactive display resolution selection
- USB tablet and keyboard support for better input handling
- Close the window to quit