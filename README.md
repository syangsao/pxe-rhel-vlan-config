# PXE Boot RHEL & OpenShift Bare Metal

PXE boot configurations for RHEL over tagged VLANs and OpenShift Container Platform on bare metal via PXE.

## Contents

| File | Description |
|------|-------------|
| [full-config.md](./full-config.md) | RHEL PXE boot with VLAN tagging — complete reference config (13 sections) |
| [openshift-baremetal-pxe-guide.md](./openshift-baremetal-pxe-guide.md) | OCP 4.22 bare metal PXE installation — end-to-end guide |

---

## RHEL PXE Boot with Tagged VLAN

End-to-end PXE network boot stack for deploying RHEL over a tagged VLAN, with MTU control at every layer — from the switch through DHCP, iPXE, kickstart, and post-install persistence.

### Architecture

```
Switch (VLAN 100 trunk)
  │
  ├── PXE/DHCP/TFTP Server (enp0s3.100) — 10.100.0.1/24
  │
  └── Target RHEL host (access port VLAN 100, or trunk with iPXE)
```

### MTU Configuration Layers

| Layer | Setting | Config |
|-------|---------|--------|
| Server VLAN interface | `mtu 9000` | NetworkManager / netplan |
| DHCP server | `option interface-mtu 9000` | `dhcpd.conf` |
| iPXE client | `set mtu-size 9000` | `boot.ipxe` |
| Kickstart | `--mtu=9000` | `rhel9.cfg` |
| Post-install | `mtu=9000` | NetworkManager connection |
| Switch | `system mtu jumbo 9216` | Cisco IOS/NX-OS |

### Quick Start

1. Read [full-config.md](./full-config.md)
2. Adjust VLAN ID, subnet, MTU, and RHEL version for your environment
3. Configure server-side: network, DHCP, TFTP, HTTP
4. Configure switch ports and MTU
5. Boot target host

---

## OpenShift Container Platform 4.22 — Bare Metal PXE

Complete bare metal PXE installation guide following Red Hat's user-provisioned infrastructure model. Covers infrastructure provisioning through cluster formation.

### Guide Sections

| # | Section |
|---|---------|
| 1 | Prerequisites — hardware, software, network requirements |
| 2 | Network architecture — topology, VLAN tagging |
| 3 | DHCP server — ISC DHCP and dnsmasq with PXE boot, VLAN, MTU |
| 4 | TFTP/PXE server — directory structure, PXELINUX config |
| 5 | HTTP server — nginx for RHCOS artifacts, Ignition configs, iPXE |
| 6 | DNS — forward/reverse zones, validation |
| 7 | Load balancer — HAProxy for API, MCS, ingress |
| 8 | OpenShift install-config.yaml — complete with networkConfig |
| 9 | Manifests & Ignition generation — `openshift-install` workflow |
| 10 | PXE boot config — iPXE script with VLAN + MTU, PXELINUX, GRUB UEFI |
| 11 | MTU configuration — switch, DHCP, iPXE, kernel args, NMState |
| 12 | VLAN configuration — switch, DHCP sub-if, iPXE VLAN, NMState |
| 13 | Network config phases — PXE boot, Ignition, Machine Config Operator |
| 14 | Installing RHCOS via PXE — bootstrap, masters, workers |
| 15 | Running the installation — `openshift-install install` |
| 16 | Post-installation — remove bootstrap, scale workers, apply MachineConfigs |
| 17 | Troubleshooting — PXE, Ignition, network, LB, stuck installation |

### Architecture

```
                    ┌─────────────────┐
                    │   Core Switch   │
                    │  (VLAN 100)     │
                    └────┬────────────┘
                         │ VLAN 100
              ┌──────────┼──────────┐
              │          │          │
        ┌─────┴──┐ ┌────┴────┐ ┌───┴─────┐
        │ DHCP   │ │ TFTP/   │ │ HTTP    │
        │ Server │ │ iPXE    │ │ Server  │
        │ 67/68  │ │ 69      │ │ 80      │
        └────────┘ └─────────┘ └─────────┘
              │          │          │
              └──────────┼──────────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
        ┌─────┴──┐ ┌────┴────┐ ┌───┴─────┐
        │Bootstrap│ │Master 1 │ │Master 2 │
        │192.168.│ │192.168. │ │192.168. │
        │1.96    │ │1.97     │ │1.98     │
        └────────┘ └─────────┘ └─────────┘
```

### Reference

- Red Hat Documentation: https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/installing_on_bare_metal/index

---

## Verification

```bash
# Test jumbo frames end-to-end:
ping -M do -s 8972 <target-ip>
# 8972 + 28 (IP+ICMP header) = 9000. Failure = MTU mismatch.
```

## Requirements

- PXE/DHCP/TFTP server (RHEL, CentOS, Ubuntu, or Debian)
- RHEL ISO (9.x) for RHEL deployments
- OpenShift installer (`openshift-install`) for OCP deployments
- Switch with VLAN support
- Target hardware with PXE-capable NIC

## License

MIT
