# PXE Boot RHEL with Tagged VLAN

Complete PXE boot configuration for RHEL with 802.1Q VLAN tagging and configurable MTU (including jumbo frames).

## What This Covers

End-to-end PXE network boot stack for deploying RHEL over a tagged VLAN, with MTU control at every layer — from the switch through DHCP, iPXE, kickstart, and post-install persistence.

## Contents

| File | Description |
|------|-------------|
| [full-config.md](./full-config.md) | Complete reference configuration — all 13 sections |

## Architecture

```
Switch (VLAN 100 trunk)
  │
  ├── PXE/DHCP/TFTP Server (enp0s3.100) — 10.100.0.1/24
  │
  └── Target RHEL host (access port VLAN 100, or trunk with iPXE)
```

## Configuration Layers

| Layer | MTU Setting | Config File |
|-------|-------------|-------------|
| Server VLAN interface | `mtu 9000` | NetworkManager / netplan |
| DHCP server | `option interface-mtu 9000` | `dhcpd.conf` |
| iPXE client | `set mtu-size 9000` | `boot.ipxe` |
| Kickstart | `--mtu=9000` | `rhel9.cfg` |
| Post-install | `mtu=9000` | NetworkManager connection |
| Switch | `system mtu jumbo 9216` | Cisco IOS/NX-OS |

## Quick Start

1. Read [full-config.md](./full-config.md)
2. Adjust VLAN ID, subnet, MTU, and RHEL version for your environment
3. Configure server-side: network, DHCP, TFTP, HTTP
4. Configure switch ports and MTU
5. Boot target host

## Verification

```bash
# Test jumbo frames end-to-end:
ping -M do -s 8972 <target-ip>
# 8972 + 28 (IP+ICMP header) = 9000. Failure = MTU mismatch.
```

## Requirements

- PXE/DHCP/TFTP server (RHEL, CentOS, Ubuntu, or Debian)
- RHEL ISO (9.x)
- Switch with VLAN support
- Target hardware with PXE-capable NIC

## License

MIT
