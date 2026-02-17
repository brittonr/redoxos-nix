# ULTRA Mode Research: Redox virtio-netd Critical Bugs

**Date**: 2026-02-17
**Issue**: Guest cannot receive inbound network packets (host->guest ping fails, TCP data doesn't arrive)

## Symptoms Observed

- Guest outbound works: ping to gateway succeeds, TCP SYN sent successfully
- Guest inbound fails: host ping to guest times out, TCP data in established connection never arrives
- TCP 3-way handshake completes (SYN-ACK works both ways) but application data doesn't flow
- ARP requests from host are not answered (required static ARP entry)

## Root Cause Analysis

### Bug #1: RX Buffers Never Recycled (CRITICAL)

**File**: `drivers/net/virtio-netd/src/scheme.rs`
**Lines**: 55-81 (`try_recv()`)

The `try_recv()` function reads packets from the virtio used ring but **never re-posts the buffer to the available ring**. After all ~256 pre-allocated buffers are consumed, the device has no buffers to write incoming packets to. They are silently dropped.

```rust
fn try_recv(&mut self, target: &mut [u8]) -> usize {
    // ... reads packet from used ring ...
    target[..payload_size].copy_from_slice(&packet);
    self.recv_head = self.rx.used.head_index();
    payload_size
    // BUG: Missing rx.send(chain) to recycle the buffer!
}
```

**Impact**: After 256 packets, all inbound traffic is dropped.

### Bug #2: Used Ring Index Tracking Skips Packets

**File**: `drivers/net/virtio-netd/src/scheme.rs`
**Lines**: 63-64, 79

```rust
let idx = self.rx.used.head_index() as usize;
let element = self.rx.used.get_element_at(idx - 1);  // Only reads LAST element
// ...
self.recv_head = self.rx.used.head_index();  // Jumps to head, skipping intermediate
```

If multiple packets arrive between calls to `try_recv()`, only the most recent one is read. Earlier packets in the used ring are never processed.

**Impact**: Packet loss under any amount of traffic burst.

### Bug #3: No IRQ-to-Event Notification for RX

The virtio MSI-X interrupt thread only wakes async futures (used for TX). The RX path doesn't use async futures, so interrupts for received packets don't wake the driver event loop. The driver only checks for packets when smolnetd makes a read() syscall.

## Research Sources

- smoltcp Device trait documentation
- Kagi search: smoltcp virtio-net issues
- Redox smolnetd source analysis (netstack/)
- Redox virtio-netd driver source analysis
- Cloud Hypervisor virtio-net documentation
- Red Hat virtio networking deep dive
- OSDev forums virtio-net troubleshooting

## Fix Required

The `try_recv()` function must:
1. Process ALL entries between `recv_head` and `used.head_index()`, not just the last one
2. After reading each packet, re-post the descriptor to the available ring via `rx.send(chain)`

## Verification

After fix:
- `ping 172.16.0.2` from host should succeed
- `nc 172.16.0.2 8023` should connect and exchange data
- TCP connections should transmit data bidirectionally
