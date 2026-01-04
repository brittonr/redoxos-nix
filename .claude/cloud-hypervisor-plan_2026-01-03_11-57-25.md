# RedoxOS on Cloud Hypervisor - Implementation Plan

**Created**: 2026-01-03 11:57:25
**Status**: Planning Phase

## Executive Summary

This document outlines the plan to run RedoxOS on [Cloud Hypervisor](https://www.cloudhypervisor.org/), a Rust-based Virtual Machine Monitor designed for modern cloud workloads. Cloud Hypervisor offers faster boot times, reduced attack surface, and better alignment with RedoxOS's Rust-based microkernel philosophy compared to QEMU.

## Research Findings

### Cloud Hypervisor Characteristics

| Feature | Cloud Hypervisor | QEMU (Current) |
|---------|-----------------|----------------|
| Language | Rust | C |
| Boot Time | <100ms to userspace | Slower (full emulation) |
| Device Model | Virtio-only (minimal) | Full legacy + virtio |
| UEFI Support | Yes (CLOUDHV.fd) | Yes (OVMF.fd) |
| IDE Support | **No** | Yes |
| virtio-blk | Yes | Yes |
| virtio-net | Yes | Yes |
| e1000 | **No** | Yes |

### RedoxOS Current Boot Requirements

1. **Firmware**: UEFI (OVMF.fd)
2. **Bootloader**: BOOTX64.EFI (custom UEFI bootloader)
3. **Storage**: IDE interface (`-drive if=ide`)
4. **Network**: Intel e1000 (`-device e1000`)
5. **Disk Format**: Raw GPT with ESP + RedoxFS partitions

### Compatibility Analysis

#### Compatible Components
- UEFI boot (Cloud Hypervisor supports CLOUDHV.fd/CLOUDHV_EFI.fd)
- Raw disk images (supported via `--disk path=`)
- GPT partition tables (standard)
- virtio-blk driver (RedoxOS has `virtio-blkd`)
- virtio-net driver (RedoxOS has `virtio-netd`)

#### Incompatible Components (Require Changes)
1. **IDE Storage**: Cloud Hypervisor does NOT support IDE
   - Solution: Switch to virtio-blk
2. **e1000 Network**: Cloud Hypervisor does NOT support e1000
   - Solution: Switch to virtio-net
3. **Bootloader loading**: Cloud Hypervisor `--kernel` expects ELF/vmlinux, not EFI
   - Solution: Use `--firmware` for UEFI boot path

## Implementation Plan

### Phase 1: Verify virtio Driver Functionality

Before switching to Cloud Hypervisor, verify RedoxOS virtio drivers work in QEMU:

```bash
# Test with virtio-blk instead of IDE
qemu-system-x86_64 \
  -bios OVMF.fd \
  -kernel BOOTX64.EFI \
  -drive file=redox.img,format=raw,if=virtio \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0 \
  ...
```

### Phase 2: Cloud Hypervisor Package

Add Cloud Hypervisor to Nix packages:

```nix
# nix/pkgs/infrastructure/cloud-hypervisor.nix
{ pkgs }:

pkgs.cloud-hypervisor or pkgs.stdenv.mkDerivation {
  pname = "cloud-hypervisor";
  version = "45.0";
  # ... build from source or use binary
}
```

### Phase 3: UEFI Firmware

Obtain Cloud Hypervisor's UEFI firmware (CLOUDHV.fd):

Options:
1. Download from [edk2 releases](https://github.com/cloud-hypervisor/edk2/releases)
2. Build from source (see docs/uefi.md)

### Phase 4: Create Cloud Hypervisor Runner

```nix
# nix/pkgs/infrastructure/cloud-hypervisor-runners.nix
runCloudHypervisor = pkgs.writeShellScriptBin "run-redox-ch" ''
  cloud-hypervisor \
    --kernel /path/to/CLOUDHV.fd \
    --disk path=$IMAGE \
    --cpus boot=4 \
    --memory size=2048M \
    --net tap=,mac=,ip=,mask= \
    --serial tty \
    --console off
'';
```

### Phase 5: Network Configuration

Cloud Hypervisor requires TAP interface setup:

```bash
# Create TAP interface (requires root or cap_net_admin)
sudo ip tuntap add dev tap0 mode tap
sudo ip link set tap0 up
sudo ip addr add 172.16.0.1/24 dev tap0

# Or use setcap for unprivileged operation
sudo setcap cap_net_admin+ep ./cloud-hypervisor
```

## Detailed Implementation Steps

### Step 1: Test virtio in QEMU First

Create a QEMU test configuration that uses virtio devices:

```nix
testVirtio = pkgs.writeShellScriptBin "test-virtio" ''
  qemu-system-x86_64 \
    -M pc \
    -cpu host \
    -m 2048 \
    -enable-kvm \
    -bios "$OVMF" \
    -kernel ${bootloader}/boot/EFI/BOOT/BOOTX64.EFI \
    -drive file="$IMAGE",format=raw,if=none,id=disk0 \
    -device virtio-blk-pci,drive=disk0 \
    -netdev user,id=net0 \
    -device virtio-net-pci,netdev=net0 \
    -serial mon:stdio \
    -nographic
'';
```

### Step 2: Add Cloud Hypervisor Package

```nix
# In flake.nix inputs
cloud-hypervisor-src = {
  url = "github:cloud-hypervisor/cloud-hypervisor/v45.0";
  flake = false;
};

# Or use nixpkgs version
cloud-hypervisor = pkgs.cloud-hypervisor;
```

### Step 3: CLOUDHV Firmware Package

```nix
cloudhvFirmware = pkgs.stdenv.mkDerivation {
  pname = "cloudhv-firmware";
  version = "edk2-stable202411";

  src = pkgs.fetchurl {
    url = "https://github.com/cloud-hypervisor/edk2/releases/download/edk2-stable202411-ch/CLOUDHV.fd";
    sha256 = "...";
  };

  installPhase = ''
    mkdir -p $out/share/firmware
    cp $src $out/share/firmware/CLOUDHV.fd
  '';
};
```

### Step 4: Cloud Hypervisor Runner Script

```nix
runCloudHypervisor = pkgs.writeShellScriptBin "run-redox-cloud-hypervisor" ''
  set -e

  WORK_DIR=$(mktemp -d)
  trap "rm -rf $WORK_DIR" EXIT

  IMAGE="$WORK_DIR/redox.img"
  cp ${diskImage}/redox.img "$IMAGE"
  chmod +w "$IMAGE"

  echo "Starting Redox OS with Cloud Hypervisor..."
  echo ""
  echo "Requirements:"
  echo "  - KVM enabled (/dev/kvm accessible)"
  echo "  - TAP networking (or run with sudo)"
  echo ""

  ${cloud-hypervisor}/bin/cloud-hypervisor \
    --kernel ${cloudhvFirmware}/share/firmware/CLOUDHV.fd \
    --disk path="$IMAGE" \
    --cpus boot=4 \
    --memory size=2048M \
    --serial tty \
    --console off
'';
```

## Risk Assessment

### High Risk
- **Bootloader compatibility**: RedoxOS bootloader may need modifications for Cloud Hypervisor's UEFI environment
- **virtio driver bugs**: Less testing of virtio drivers compared to IDE/e1000

### Medium Risk
- **Network setup complexity**: TAP interfaces require elevated privileges
- **Firmware differences**: CLOUDHV.fd may have subtle differences from OVMF.fd

### Low Risk
- **Disk image format**: Raw GPT images are fully compatible
- **Serial console**: Both hypervisors support standard serial

## Rollback Strategy

If Cloud Hypervisor doesn't work:
1. Keep QEMU runners as default
2. Make Cloud Hypervisor runner optional/experimental
3. Document known issues for future fixes

## Testing Plan

1. **Unit Test**: Boot with virtio in QEMU first
2. **Integration Test**: Boot with Cloud Hypervisor + CLOUDHV.fd
3. **Network Test**: Verify virtio-net connectivity
4. **Storage Test**: Verify virtio-blk read/write operations
5. **Boot Test**: Full boot to shell with Cloud Hypervisor

## Monitoring & Metrics

Track:
- Boot time comparison (QEMU vs Cloud Hypervisor)
- Memory usage comparison
- Network throughput with virtio-net
- Storage performance with virtio-blk

## Implementation Status (2026-01-03)

### Completed Fixes
1. [x] Test virtio-blk and virtio-net in QEMU - SUCCESS
2. [x] Package Cloud Hypervisor in Nix - Uses nixpkgs#cloud-hypervisor
3. [x] Package CLOUDHV.fd firmware - Uses nixpkgs#OVMF-cloud-hypervisor.fd
4. [x] Create Cloud Hypervisor runner script - `nix/pkgs/infrastructure/cloud-hypervisor-runners.nix`
5. [x] **Fix PCI BAR relocation issue** - Added `read_bar_no_probe()` in pcid to skip BAR size probing for virtio devices
6. [x] **Add modern virtio device IDs** - Added device ID 0x1042 (virtio-blk) and 0x1041 (virtio-net) to initfs.toml
7. [x] **Fix virtio driver device ID assertions** - Updated virtio-blkd and virtio-netd to accept both legacy and modern device IDs
8. [x] **Fix BAR size for MSI-X** - Increased default BAR size to 1MB for MSI-X tables and PBA structures
9. [x] **Fix queue modulo bug** - Fixed `% 256` to `% queue_size` in virtio-core/src/transport.rs:434
10. [x] **Add is_modern_virtio() helper** - Added function to detect modern virtio devices by device ID (0x1040-0x107F)
11. [x] **Fix modern virtio byte count handling** - Updated virtio-blkd to handle modern virtio's different byte count reporting

### Current Status: FULLY WORKING

**QEMU with virtio devices**: Full success
- RedoxOS boots completely with virtio-blk and virtio-net
- Shell works, ping works

**Cloud Hypervisor**: FULLY WORKING (as of 2026-01-03)
- Bootloader: WORKS - Finds RedoxFS via device path partition info
- Kernel: WORKS - Loads and starts
- PCI BAR: FIXED - No more relocation errors
- MSI-X: FIXED - Tables and PBA accessible
- Device ID: FIXED - Modern virtio IDs recognized (0x1042)
- Queue modulo: FIXED - No more 256 hardcoded limit
- Byte count: FIXED - is_modern flag properly differentiates QEMU vs CH
- Partition offset: FIXED - Reads from UEFI device path instead of hardcoded 2 MiB
- RedoxFS mount: WORKS - Successfully mounts and transitions to root filesystem
- Shell: WORKS - Ion shell runs and displays help

### Fixed Issue: Modern Virtio Protocol

The 511 vs 512 assertion failure was caused by two bugs:

1. **Queue modulo bug**: `virtio-core/src/transport.rs:434` had `% 256` hardcoded instead of `% queue_size`
2. **Byte count mismatch**: Modern virtio returns data bytes only, legacy includes status byte

Fix applied:
- `is_modern_virtio(device_id)` function detects modern devices (0x1040-0x107F)
- virtio-blkd conditionally handles byte count based on device type

### Files Modified

**In `redox-src/recipes/core/base/source/`:**

1. `drivers/pcid/src/main.rs`:
   - Added `read_bar_no_probe()` - reads BARs without size probing
   - Added `is_virtio_device()` - checks for vendor 0x1AF4
   - Modified `handle_parsed_header()` to use no-probe for virtio devices
   - Skipped ROM probing for virtio devices
   - Set default BAR size to 1MB for MSI-X compatibility

2. `drivers/virtio-core/src/lib.rs`:
   - Added `is_modern_virtio(device_id)` helper function

3. `drivers/virtio-core/src/transport.rs`:
   - Fixed `get_mut_element_at()` to use `% queue_size` instead of `% 256`

4. `drivers/storage/virtio-blkd/src/main.rs`:
   - Updated device ID assertion to accept 0x1001 (legacy) and 0x1042 (modern)
   - Added modern device detection using `is_modern_virtio()`

5. `drivers/storage/virtio-blkd/src/scheme.rs`:
   - Added `is_modern` flag to VirtioDisk struct
   - Updated read/write to conditionally handle byte count for modern vs legacy
   - Modern virtio (Cloud Hypervisor) returns data bytes only
   - Legacy virtio (QEMU) includes status byte in count

6. `drivers/net/virtio-netd/src/main.rs`:
   - Updated device ID assertion to accept 0x1000 (legacy) and 0x1041 (modern)

**In `nix/pkgs/infrastructure/`:**

1. `initfs.nix`:
   - Added modern virtio-blk device ID (0x1042) to driver config
   - Added modern virtio-net device ID (0x1041) to driver config

**In `redox-src/recipes/core/bootloader/source/src/os/uefi/`:**

1. `device.rs`:
   - Added `get_partition_start_from_device_path()` helper to extract partition offset from UEFI device path
   - Added `get_redoxfs_offset_from_gpt_buffer()` helper for live images
   - Updated `disk_device_priority()` to use device path info instead of hardcoded 2 MiB offset
   - Fixed live image partition offset detection

2. `mod.rs`:
   - Added diagnostic logging for partition offset being used

### Cloud Hypervisor Support Complete

Cloud Hypervisor now fully boots RedoxOS:
```bash
nix run .#run-redox-cloud-hypervisor
```

Expected output includes:
- "Looking for RedoxFS:" with HD(0x2,GPT,...) partition
- "Mounting RedoxFS..." with no errors
- "Redox OS Boot Complete!"
- Ion shell help output

## How to Use

### QEMU (Working)
```bash
nix run .#run-redox              # Headless with virtio devices
nix run .#run-redox-graphical    # Graphical mode
```

### Cloud Hypervisor (Experimental)
```bash
nix run .#run-redox-cloud-hypervisor        # Basic boot (root mount issues)
nix run .#run-redox-cloud-hypervisor-net    # With TAP networking (requires setup)
```

## Sources

- [Cloud Hypervisor Quick Start](https://www.cloudhypervisor.org/docs/prologue/quick-start/)
- [Cloud Hypervisor UEFI Documentation](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/uefi.md)
- [Cloud Hypervisor GitHub](https://github.com/cloud-hypervisor/cloud-hypervisor)
- [Rust Hypervisor Firmware](https://github.com/cloud-hypervisor/rust-hypervisor-firmware)
- [EDK2 CLOUDHV Releases](https://github.com/cloud-hypervisor/edk2/releases)
