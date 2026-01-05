# Cloud Hypervisor Implementation Improvements

**Created**: 2026-01-05 20:20:35
**Status**: Recommendations Ready for Implementation

## Executive Summary

Based on comprehensive research of Cloud Hypervisor v49-50 features, best practices, and analysis of the current RedoxOS implementation, this document outlines concrete improvements organized by priority.

## Current Implementation Analysis

### Strengths
- Working headless and networked runners
- Proper TAP interface setup with NAT
- Modern virtio device ID patches for RedoxOS drivers
- Two disk image variants (DHCP/static)
- Comprehensive error handling

### Gaps Identified
1. No performance tuning options (io_uring, huge pages)
2. Single-queue virtio-net (no multi-queue)
3. No API socket for runtime control
4. No snapshot/restore capability
5. No huge pages support
6. Missing CPU topology configuration
7. No rate limiting/QoS options

## Prioritized Improvement Plan

### PRIORITY 1: Performance Tuning (High Impact, Low Effort)

#### 1.1 Storage Performance with io_uring

**Current**:
```bash
--disk path="$IMAGE"
```

**Improved**:
```bash
--disk path="$IMAGE",direct=on
```

**Rationale**: io_uring is enabled by default in Cloud Hypervisor, but `direct=on` bypasses host page cache for better I/O performance. Provides 60% improvement in single-queue throughput with EVENT_IDX.

**Implementation**: Modify `cloud-hypervisor-runners.nix` lines 187 and 260.

#### 1.2 Multi-Queue Networking

**Current**:
```bash
--net tap="$TAP_NAME",mac="$GUEST_MAC"
```

**Improved**:
```bash
--net tap="$TAP_NAME",mac="$GUEST_MAC",num_queues=4,queue_size=256
```

**Rationale**: Multi-queue enables parallel packet processing across vCPUs, significantly improving network throughput for multi-core guests.

**Implementation**: Modify line 265.

#### 1.3 CPU Topology

**Current**:
```bash
--cpus boot=4
```

**Improved**:
```bash
--cpus boot=4,topology=1:2:2
```

**Rationale**: Proper socket/core/thread topology helps guest OS scheduler make better decisions.

**Implementation**: Modify lines 188 and 261.

### PRIORITY 2: Runtime Control (Medium Impact, Medium Effort)

#### 2.1 API Socket Support

Add optional API socket for runtime VM control:

```nix
# New parameter in cloud-hypervisor-runners.nix
apiSocket ? "/tmp/cloud-hypervisor-redox.sock",

# Add to cloud-hypervisor command
--api-socket path="${apiSocket}"
```

**Benefits**:
- Pause/resume VM
- Snapshot/restore (offline migration, testing)
- Memory/CPU hotplug
- Dynamic disk addition
- Runtime monitoring

#### 2.2 ch-remote Integration

Provide wrapper scripts:

```nix
# New script: pause-redox
pauseRedox = pkgs.writeShellScriptBin "pause-redox" ''
  ${pkgs.cloud-hypervisor}/bin/ch-remote \
    --api-socket=/tmp/cloud-hypervisor-redox.sock pause
'';

# New script: snapshot-redox
snapshotRedox = pkgs.writeShellScriptBin "snapshot-redox" ''
  DEST="''${1:-/tmp/redox-snapshot}"
  ${pkgs.cloud-hypervisor}/bin/ch-remote \
    --api-socket=/tmp/cloud-hypervisor-redox.sock pause
  ${pkgs.cloud-hypervisor}/bin/ch-remote \
    --api-socket=/tmp/cloud-hypervisor-redox.sock \
    snapshot "file://$DEST"
  echo "Snapshot saved to $DEST"
'';
```

### PRIORITY 3: Memory Optimization (Medium Impact, Medium Effort)

#### 3.1 Huge Pages Support

**Current**:
```bash
--memory size=2048M
```

**Improved**:
```bash
--memory size=2048M,hugepages=on
```

**Prerequisites**: Host must have huge pages allocated:
```bash
# On host (one-time setup)
echo 1024 > /proc/sys/vm/nr_hugepages
# Or in NixOS config:
boot.kernelParams = [ "hugepagesz=2M" "hugepages=1024" ];
```

**Fallback**: Make hugepages optional via environment variable:
```bash
MEMORY_OPTS="size=2048M"
if [ -n "$CH_HUGEPAGES" ]; then
  MEMORY_OPTS="size=2048M,hugepages=on"
fi
--memory $MEMORY_OPTS
```

#### 3.2 Memory Hotplug with virtio-mem

For development/testing scenarios:
```bash
--memory size=2048M,hugepages=on,hotplug_method=virtio-mem,hotplug_size=1024M
```

Allows runtime memory expansion up to 3GB without guest reboot.

### PRIORITY 4: Advanced Networking (Lower Impact, Higher Effort)

#### 4.1 Rate Limiting/QoS

Add I/O throttling options:
```bash
--net tap="$TAP_NAME",mac="$GUEST_MAC",bw_size=10485760,bw_refill_time=100
# 10MB/100ms = 100 MB/s bandwidth limit
```

#### 4.2 MACVTAP Alternative

For improved performance without NAT complexity:
```nix
withMacvtap = pkgs.writeShellScriptBin "run-redox-cloud-hypervisor-macvtap" ''
  # MACVTAP provides direct bridge to host network
  # Setup requires: ip link add link eth0 name macvtap0 type macvtap mode bridge

  MACVTAP_DEV="''${MACVTAP_DEV:-macvtap0}"

  if ! ip link show "$MACVTAP_DEV" &>/dev/null; then
    echo "MACVTAP interface $MACVTAP_DEV not found!"
    echo "Create with: sudo ip link add link eth0 name macvtap0 type macvtap mode bridge"
    exit 1
  fi

  # Get tap device fd
  TAP_INDEX=$(cat /sys/class/net/$MACVTAP_DEV/ifindex)
  exec 3< /dev/tap$TAP_INDEX

  ${cloudHypervisor}/bin/cloud-hypervisor \
    --firmware "$FIRMWARE" \
    --disk path="$IMAGE",direct=on \
    --cpus boot=4,topology=1:2:2 \
    --memory size=2048M \
    --net fd=3,mac="52:54:00:12:34:56" \
    --serial tty \
    --console off
'';
```

### PRIORITY 5: Developer Experience (Low Impact, Low Effort)

#### 5.1 Profile-Based Configuration

Create multiple runner profiles:
```nix
runners = {
  # Minimal (default, fast startup)
  headless = mkRunner { profile = "minimal"; };

  # Performance (optimized I/O, networking)
  performance = mkRunner {
    profile = "performance";
    directIO = true;
    numQueues = 4;
    hugepages = true;
  };

  # Development (with API socket and debugging)
  dev = mkRunner {
    profile = "development";
    apiSocket = true;
    debugSerial = true;
  };
};
```

#### 5.2 Environment Variable Overrides

Document and support:
```bash
# Already supported:
TAP_NAME=tap1 nix run .#run-redox-cloud-hypervisor-net

# Add support for:
CH_CPUS=8 \
CH_MEMORY=4096M \
CH_HUGEPAGES=on \
CH_API_SOCKET=/tmp/my-vm.sock \
  nix run .#run-redox-cloud-hypervisor
```

## Implementation Roadmap

### Phase 1: Quick Wins (1-2 hours)
1. Add `direct=on` to disk configuration
2. Add `num_queues=4,queue_size=256` to network
3. Add `topology=1:2:2` to CPU configuration
4. Update documentation

### Phase 2: API Integration (2-4 hours)
1. Add optional `--api-socket` parameter
2. Create ch-remote wrapper scripts
3. Add pause/resume/snapshot apps
4. Test snapshot/restore workflow

### Phase 3: Memory Optimization (2-4 hours)
1. Add hugepages option (environment-controlled)
2. Add virtio-mem hotplug support
3. Document host prerequisites
4. Test memory expansion

### Phase 4: Advanced Features (4-8 hours)
1. Add rate limiting options
2. Create MACVTAP runner variant
3. Add profile-based configuration
4. Comprehensive documentation

## Concrete Code Changes

### cloud-hypervisor-runners.nix (Improved Version)

```nix
# Add to let block
cpuTopology = "1:2:2";  # 1 socket, 2 cores, 2 threads
netQueues = 4;
netQueueSize = 256;

# Headless runner changes (lines 185-194)
${cloudHypervisor}/bin/cloud-hypervisor \
  --firmware "$FIRMWARE" \
  --disk path="$IMAGE",direct=on \
  --cpus boot=4,topology=${cpuTopology} \
  --memory size=2048M \
  --platform num_pci_segments=1 \
  --pci-segment pci_segment=0,mmio32_aperture_weight=4 \
  --serial tty \
  --console off \
  "$@"

# Network runner changes (lines 258-268)
${cloudHypervisor}/bin/cloud-hypervisor \
  --firmware "$FIRMWARE" \
  --disk path="$IMAGE",direct=on \
  --cpus boot=4,topology=${cpuTopology} \
  --memory size=2048M \
  --platform num_pci_segments=1 \
  --pci-segment pci_segment=0,mmio32_aperture_weight=4 \
  --net tap="$TAP_NAME",mac="$GUEST_MAC",num_queues=${toString netQueues},queue_size=${toString netQueueSize} \
  --serial tty \
  --console off \
  "$@"
```

## Security Considerations

### Already Implemented
- Seccomp filtering (built into Cloud Hypervisor)
- Rust memory safety
- Minimal PCI segments
- Disabled virtio-console

### Recommended Additions
- Add `--seccomp log` for audit logging during development
- Consider `--landlock` when available in future versions
- Document security model for production deployments

## Testing Checklist

After implementing improvements:

- [ ] Boot test with direct I/O
- [ ] Network throughput test with multi-queue
- [ ] CPU topology verified in guest (`lscpu`)
- [ ] API socket functionality (pause/resume)
- [ ] Snapshot/restore workflow
- [ ] Memory hotplug (if enabled)
- [ ] Huge pages allocation (if enabled)
- [ ] Rate limiting verification
- [ ] Error handling for missing prerequisites

## Sources

- [Cloud Hypervisor GitHub](https://github.com/cloud-hypervisor/cloud-hypervisor)
- [Cloud Hypervisor v50 Release](https://github.com/cloud-hypervisor/cloud-hypervisor/releases/tag/v50.0)
- [Cloud Hypervisor I/O Throttling](https://intelkevinputnam.github.io/cloud-hypervisor-docs-HTML/docs/io_throttling.html)
- [Cloud Hypervisor Snapshot/Restore](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/snapshot_restore.md)
- [Cloud Hypervisor Hotplug](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/hotplug.md)
- [Phoronix: Cloud Hypervisor 0.9 io_uring](https://www.phoronix.com/news/Cloud-Hypervisor-0.9)
- [Red Hat: Optimizing Virtual Machine Performance](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_virtualization/optimizing-virtual-machine-performance-in-rhel_configuring-and-managing-virtualization)
