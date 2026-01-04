# Cloud Hypervisor Networking for RedoxOS

**Created**: 2026-01-03 15:04:03
**Status**: Implemented

## Overview

This document describes the complete networking setup for running RedoxOS in Cloud Hypervisor with full network connectivity. Unlike QEMU's user-mode networking (SLIRP), Cloud Hypervisor requires TAP-based networking with host-side NAT configuration.

## Architecture

```
+------------------+      TAP Interface     +------------------+
|    Host Linux    |  <-------------------> |     RedoxOS      |
|  172.16.0.1/24   |        tap0            |   172.16.0.2/24  |
+------------------+                        +------------------+
        |
        | NAT (iptables masquerade)
        v
   Internet
```

### Network Configuration

| Component | Value |
|-----------|-------|
| Host TAP IP | 172.16.0.1/24 |
| Guest IP | 172.16.0.2/24 |
| Guest Gateway | 172.16.0.1 |
| Guest MAC | 52:54:00:12:34:56 |
| TAP Interface | tap0 |
| Subnet | 172.16.0.0/24 |

## Quick Start

### 1. Set Up Host Networking (One-Time Setup)

```bash
# Run the setup helper as root
sudo nix run .#setup-cloud-hypervisor-network
```

This script:
- Creates the TAP interface (tap0)
- Assigns IP 172.16.0.1/24 to the host
- Enables IP forwarding
- Configures NAT/masquerading for internet access

### 2. Run RedoxOS with Networking

```bash
nix run .#run-redox-cloud-hypervisor-net
```

### 3. Configure Network Inside RedoxOS

Once RedoxOS boots, configure the network:

```ion
# Run the Cloud Hypervisor network config script
/bin/netcfg-ch

# Or manually:
ifconfig network.0000:00:04.0_virtio_net 172.16.0.2 netmask 255.255.255.0
echo "172.16.0.1" > /etc/net/ip_router

# Test connectivity
ping 172.16.0.1
```

## Implementation Details

### Host-Side Components

#### TAP Setup Script (`setup-cloud-hypervisor-network`)

Location: `nix/pkgs/infrastructure/cloud-hypervisor-runners.nix`

The script:
1. Creates TAP interface owned by current user
2. Assigns 172.16.0.1/24 to the interface
3. Enables IP forwarding via `/proc/sys/net/ipv4/ip_forward`
4. Adds iptables NAT rules:
   - POSTROUTING MASQUERADE for 172.16.0.0/24
   - FORWARD rules for bidirectional traffic

#### Cloud Hypervisor Runner (`run-redox-cloud-hypervisor-net`)

Passes network configuration to Cloud Hypervisor:
```bash
cloud-hypervisor \
  --net tap="tap0",mac="52:54:00:12:34:56" \
  ...
```

### Guest-Side Components

#### Network Configuration Files

Located in disk image at:
- `/etc/net/cloud-hypervisor/ip` - Guest IP address (172.16.0.2)
- `/etc/net/cloud-hypervisor/netmask` - Netmask (255.255.255.0)
- `/etc/net/cloud-hypervisor/gateway` - Gateway (172.16.0.1)
- `/etc/net/dns` - DNS servers (1.1.1.1, 8.8.8.8)

#### Network Configuration Script (`/bin/netcfg-ch`)

Ion shell script that:
1. Finds the virtio-net interface in `/scheme`
2. Reads configuration from `/etc/net/cloud-hypervisor/`
3. Configures interface with `ifconfig`
4. Sets gateway in `/etc/net/ip_router`

### Driver Stack

```
Application
    |
    v
smolnetd (network stack daemon)
    |
    v
virtio-netd (network driver)
    |
    v
Cloud Hypervisor (virtio-net device)
    |
    v
TAP interface (host)
```

#### virtio-netd Driver

Location: `redox-src/recipes/core/base/source/drivers/net/virtio-netd/`

Supports:
- Legacy device ID: 0x1000 (QEMU)
- Modern device ID: 0x1041 (Cloud Hypervisor)

#### smolnetd

Location: `redox-src/recipes/core/base/source/netstack/`

Provides:
- `/scheme/ip` - Raw IP sockets
- `/scheme/tcp` - TCP sockets
- `/scheme/udp` - UDP sockets
- `/scheme/icmp` - ICMP (ping)
- `/scheme/netcfg` - Network configuration

## Manual Host Setup

If you prefer manual setup instead of the helper script:

```bash
# Create TAP interface
sudo ip tuntap add dev tap0 mode tap user $USER

# Bring interface up
sudo ip link set tap0 up

# Assign IP address
sudo ip addr add 172.16.0.1/24 dev tap0

# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Set up NAT (replace eth0 with your internet interface)
sudo iptables -t nat -A POSTROUTING -s 172.16.0.0/24 -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i tap0 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o tap0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## Cleanup

To remove the TAP interface:

```bash
sudo ip link delete tap0
```

To remove iptables rules:

```bash
sudo iptables -t nat -D POSTROUTING -s 172.16.0.0/24 -o eth0 -j MASQUERADE
sudo iptables -D FORWARD -i tap0 -o eth0 -j ACCEPT
sudo iptables -D FORWARD -i eth0 -o tap0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## Environment Variables

The runner scripts support customization via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `TAP_NAME` | tap0 | TAP interface name |
| `HOST_IP` | 172.16.0.1 | Host-side IP address |
| `GUEST_IP` | 172.16.0.2 | Guest IP address |
| `GUEST_MAC` | 52:54:00:12:34:56 | Guest MAC address |

Example:
```bash
TAP_NAME=vmtap0 nix run .#run-redox-cloud-hypervisor-net
```

## Troubleshooting

### TAP Interface Not Found

Error: "TAP interface tap0 not found!"

Solution: Run the setup script:
```bash
sudo nix run .#setup-cloud-hypervisor-network
```

### No Network Interface in RedoxOS

Check if virtio-netd driver loaded:
```bash
ls /scheme | grep network
```

If empty, check PCI device enumeration in boot log.

### Can't Ping Gateway

1. Verify TAP interface is up on host:
   ```bash
   ip link show tap0
   ```

2. Check IP is assigned:
   ```bash
   ip addr show tap0
   ```

3. Verify smolnetd is running in guest (check boot log)

### No Internet Access

1. Check IP forwarding:
   ```bash
   cat /proc/sys/net/ipv4/ip_forward
   # Should be 1
   ```

2. Check NAT rules:
   ```bash
   sudo iptables -t nat -L -n
   # Should show MASQUERADE rule
   ```

3. Verify default internet interface in iptables rules matches actual interface

## Comparison: QEMU vs Cloud Hypervisor Networking

| Feature | QEMU (SLIRP) | Cloud Hypervisor (TAP) |
|---------|--------------|------------------------|
| Host setup | None required | TAP + NAT required |
| Guest IP | DHCP (10.0.2.15) | Static (172.16.0.2) |
| Gateway | 10.0.2.2 | 172.16.0.1 |
| Performance | Lower (userspace) | Higher (kernel TAP) |
| Host access | Via hostfwd | Direct |
| Root required | No | Yes (for setup) |

## Files Modified

1. `nix/pkgs/infrastructure/cloud-hypervisor-runners.nix`
   - Added `setupNetwork` script for host TAP/NAT setup
   - Enhanced `withNetwork` runner with MAC address and better UX
   - Added network configuration documentation

2. `nix/flake-modules/apps.nix`
   - Added `setup-cloud-hypervisor-network` app

3. `nix/flake-modules/packages.nix`
   - Added `setupCloudHypervisorNetwork` package

4. `nix/pkgs/infrastructure/disk-image.nix`
   - Added Cloud Hypervisor network config files
   - Added `/bin/netcfg-ch` helper script

## Future Improvements

1. **Auto-detection**: Detect hypervisor type at boot and configure network automatically
2. **DHCP support**: Implement DHCP server on host for automatic guest configuration
3. **Bridge networking**: Support bridged networking for multi-VM setups
4. **IPv6**: Add IPv6 support to network configuration
5. **vhost-user**: Explore vhost-user for higher performance networking
