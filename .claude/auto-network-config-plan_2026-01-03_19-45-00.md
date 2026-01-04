# Automatic Network Configuration Plan for RedoxOS

**Created**: 2026-01-03 19:45:00
**Status**: Plan

## Problem Statement

Currently, when running RedoxOS in Cloud Hypervisor with TAP networking, the guest requires manual network configuration via `/bin/netcfg-ch`. This is inconvenient for automated testing and development workflows.

**Goal**: Automatically configure network at boot time based on detected hypervisor/environment.

## Architecture Analysis

### Current Boot Sequence

```
UEFI Bootloader → Kernel → InitFS → RootFS → Shell
                           │
                           ├── randd, ptyd, logd (daemons)
                           ├── pcid-spawner → virtio-netd/e1000d
                           ├── redoxfs mount
                           └── run.d /usr/lib/init.d /etc/init.d
                                      │
                                      ├── 00_base: ipcd
                                      ├── 10_net: smolnetd
                                      └── 15_dhcp: dhcpd
```

### Key Files

| File | Purpose |
|------|---------|
| `nix/pkgs/infrastructure/initfs.nix` | Creates initfs with drivers and init.rc |
| `nix/pkgs/infrastructure/disk-image.nix` | Creates rootfs with init.d scripts |
| `disk-image.nix:276-287` | Creates init.d scripts (00_base, 10_net, 15_dhcp) |
| `disk-image.nix:291-339` | Creates `/bin/netcfg-ch` helper script |

### Network Configuration Points

1. **DHCP (QEMU user-mode)**: Works automatically via dhcpd
2. **Static IP (Cloud Hypervisor TAP)**: Requires manual `netcfg-ch`

## Solution Options

### Option 1: Build-Time Hypervisor-Specific Images (Recommended)

Create separate disk images with different default configurations:

```nix
# Factory function approach
mkDiskImage = { networkMode ? "dhcp", ... }:
  # networkMode: "dhcp" | "cloud-hypervisor" | "static"
```

**Pros**:
- Clean separation of concerns
- No runtime detection complexity
- Reproducible builds
- Works with current RedoxOS init system

**Cons**:
- Multiple image variants to maintain
- User must choose correct image

### Option 2: Runtime Auto-Detection (Complex)

Detect hypervisor at boot and configure accordingly:

```ion
# /bin/netcfg-auto
# Detect hypervisor from PCI vendor IDs or CPUID
let hypervisor = $(detect-hypervisor)
if test "$hypervisor" = "cloud-hypervisor"
    netcfg-ch
else
    # DHCP is already started
end
```

**Pros**:
- Single image works everywhere
- User doesn't need to choose

**Cons**:
- Hypervisor detection is complex in RedoxOS
- No CPUID access from userspace currently
- Adds boot-time complexity

### Option 3: Init Script Auto-Configuration (Recommended Hybrid)

Add an auto-configuration init script that runs after smolnetd:

```
/etc/init.d/16_netcfg  # Runs after 15_dhcp
```

This script:
1. Waits for network interface to appear
2. Checks if DHCP succeeded (IP assigned)
3. If no IP, applies static Cloud Hypervisor config

**Pros**:
- Works with single image
- Falls back gracefully
- Uses existing init.d infrastructure

**Cons**:
- Slightly longer boot time
- DHCP timeout delay if not available

## Recommended Implementation: Option 3 (Hybrid)

### Phase 1: Add Auto-Configuration Init Script

Create `/etc/init.d/16_netcfg` that:

```ion
#!/bin/ion
# Auto-configure network if DHCP fails

# Wait for network interface
let timeout = 10
let iface = ""
while test $timeout -gt 0
    for entry in $(ls /scheme)
        if test "$entry" =~ "network"
            let iface = "$entry"
            break
        end
    end
    if test -n "$iface"
        break
    end
    sleep 1
    let timeout = $timeout - 1
end

if test -z "$iface"
    echo "netcfg-auto: No network interface found"
    exit 0
end

# Check if DHCP assigned an IP (wait a moment for dhcpd)
sleep 2
let has_ip = $(ifconfig "$iface" 2>/dev/null | grep -c "inet")

if test "$has_ip" = "0"
    # No DHCP response - apply static config
    if exists -f /etc/net/cloud-hypervisor/ip
        echo "netcfg-auto: Applying Cloud Hypervisor static config..."
        let ip = $(cat /etc/net/cloud-hypervisor/ip)
        let netmask = $(cat /etc/net/cloud-hypervisor/netmask)
        let gateway = $(cat /etc/net/cloud-hypervisor/gateway)
        ifconfig "$iface" "$ip" netmask "$netmask"
        echo "$gateway" > /etc/net/ip_router
        echo "netcfg-auto: Configured $iface with $ip"
    end
else
    echo "netcfg-auto: DHCP configuration detected, skipping static config"
end
```

### Phase 2: Parameterize Disk Image Build

Modify `disk-image.nix` to accept network configuration:

```nix
{
  # ... existing params ...

  # Network configuration mode
  # "auto" - Try DHCP, fallback to static (default)
  # "dhcp" - DHCP only (QEMU user-mode)
  # "static" - Static Cloud Hypervisor config only
  # "none" - No auto-configuration
  networkMode ? "auto",

  # Static IP configuration (when networkMode = "static" or "auto")
  staticNetworkConfig ? {
    ip = "172.16.0.2";
    netmask = "255.255.255.0";
    gateway = "172.16.0.1";
  },
}:
```

### Phase 3: Create Image Variants in packages.nix

```nix
# Default image with auto-detection
diskImage = modularPkgs.infrastructure.mkDiskImage {
  inherit (modularPkgs.system) kernel bootloader base;
  inherit initfs sodium;
  inherit (modularPkgs.userspace) ion uutils helix binutils extrautils netutils;
  redoxfs = modularPkgs.host.redoxfs;
  networkMode = "auto";
};

# Cloud Hypervisor optimized image
diskImageCloudHypervisor = modularPkgs.infrastructure.mkDiskImage {
  inherit (modularPkgs.system) kernel bootloader base;
  inherit initfs sodium;
  inherit (modularPkgs.userspace) ion uutils helix binutils extrautils netutils;
  redoxfs = modularPkgs.host.redoxfs;
  networkMode = "static";
  staticNetworkConfig = {
    ip = "172.16.0.2";
    netmask = "255.255.255.0";
    gateway = "172.16.0.1";
  };
};
```

### Phase 4: Update Cloud Hypervisor Runners

Modify `cloud-hypervisor-runners.nix` to use the optimized image:

```nix
{
  pkgs,
  lib,
  diskImage,           # Default image (for headless)
  diskImageNet ? null, # Network-optimized image (optional)
}:

# withNetwork uses diskImageNet if provided, else diskImage
withNetwork = pkgs.writeShellScriptBin "run-redox-cloud-hypervisor-net" ''
  IMAGE_SOURCE="${if diskImageNet != null then diskImageNet else diskImage}/redox.img"
  # ... rest of script
'';
```

## Implementation Steps

### Step 1: Modify disk-image.nix

1. Add `networkMode` and `staticNetworkConfig` parameters
2. Conditionally create `/etc/init.d/16_netcfg` based on networkMode
3. Update static config files based on `staticNetworkConfig`

### Step 2: Create netcfg-auto Script

1. Add `/bin/netcfg-auto` Ion script to disk image
2. Make it executable
3. Test with both QEMU and Cloud Hypervisor

### Step 3: Update packages.nix

1. Add `diskImageCloudHypervisor` variant
2. Update runner references
3. Add to flake packages

### Step 4: Update cloud-hypervisor-runners.nix

1. Accept optional `diskImageNet` parameter
2. Use appropriate image for network runner

### Step 5: Test and Validate

1. Test QEMU with `networkMode = "auto"` (DHCP should work)
2. Test Cloud Hypervisor with `networkMode = "auto"` (static fallback)
3. Test Cloud Hypervisor with `networkMode = "static"` (immediate static)
4. Verify ping to gateway in all scenarios

## File Changes Summary

| File | Change |
|------|--------|
| `nix/pkgs/infrastructure/disk-image.nix` | Add networkMode param, create 16_netcfg |
| `nix/pkgs/infrastructure/default.nix` | Pass new params to mkDiskImage |
| `nix/flake-modules/packages.nix` | Add diskImageCloudHypervisor |
| `nix/flake-modules/apps.nix` | (Optional) Add run-redox-ch-auto |
| `nix/pkgs/infrastructure/cloud-hypervisor-runners.nix` | Use network-optimized image |

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| DHCP timeout delays boot | Keep timeout short (2-5 seconds) |
| Ion script complexity | Test thoroughly, use simple constructs |
| ifconfig output parsing | Use grep -c for simple detection |
| Multiple image maintenance | Factory function reduces duplication |

## Success Criteria

1. Cloud Hypervisor boots with network configured automatically
2. QEMU user-mode networking still works (DHCP)
3. No manual intervention required
4. Boot time increase < 5 seconds
5. `ping 172.16.0.1` works immediately after boot

## Alternative: Kernel Command Line (Future)

For future consideration, RedoxOS could support network configuration via kernel command line:

```
net.ip=172.16.0.2 net.gateway=172.16.0.1 net.dns=1.1.1.1
```

This would require:
1. Kernel changes to parse and expose these
2. Init changes to read from kernel config
3. More invasive than init.d approach

## Timeline Estimate

| Phase | Effort |
|-------|--------|
| Phase 1: Init script | 1-2 hours |
| Phase 2: Parameterize build | 1-2 hours |
| Phase 3: Image variants | 30 min |
| Phase 4: Update runners | 30 min |
| Phase 5: Testing | 1-2 hours |
| **Total** | **4-7 hours** |
