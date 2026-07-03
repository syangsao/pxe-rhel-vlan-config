# PXE Boot RHEL with Tagged VLAN — Full Configuration

## Architecture

```
Switch (VLAN 100 trunk)
  │
  ├── PXE/DHCP/TFTP Server (enp0s3.100) — 10.100.0.1/24
  │
  └── Target RHEL host (access port VLAN 100, or trunk with iPXE)
```

All services run on a single server. Adjust hostnames, IPs, VLAN IDs, and MTU to match your environment.

---

## Configuration Variables

| Variable | Example | Description |
|----------|---------|-------------|
| `VLAN_ID` | 100 | 802.1Q VLAN tag |
| `PHYNIC` | enp0s3 | Physical NIC on PXE server |
| `VLANNIC` | enp0s3.100 | VLAN sub-interface |
| `SERVER_IP` | 10.100.0.1 | Server IP on VLAN |
| `SUBNET` | 10.100.0.0/24 | Deployment subnet |
| `MTU` | 9000 | Jumbo frame MTU (or 1500 standard) |
| `RHEL_ISO` | /var/iso/rhel-9.5-x86_64-dvd.iso | Mounted RHEL ISO |
| `KICKSTART` | /var/lib/tftpboot/ks/rhel9.cfg | Kickstart file |

---

## 1. Server Network Configuration

### 1a. Create VLAN sub-interface with MTU

**NetworkManager (RHEL/CentOS/Fedora):**

```bash
nmcli con add type vlan \
  conname vlan100 \
  ifname enp0s3.100 \
  dev enp0s3 \
  id 100 \
  ipv4.addresses 10.100.0.1/24 \
  802-3-ethernet.mtu 9000 \
  autoconnect yes
```

**netplan (Ubuntu/Debian):**

```yaml
# /etc/netplan/01-vlan.yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
  vlans:
    enp0s3.100:
      id: 100
      link: enp0s3
      dhcp4: no
      addresses:
        - 10.100.0.1/24
      mtu: 9000
```

```bash
netplan apply
```

**Traditional interfaces file (Debian/Ubuntu):**

```bash
# /etc/network/interfaces
auto enp0s3.100
iface enp0s3.100 inet static
    vlan-raw-device enp0s3
    address 10.100.0.1
    netmask 255.255.255.0
    mtu 9000
```

```bash
ifup enp0s3.100
```

### 1b. Verify

```bash
ip link show enp0s3.100
# Should show: mtu 9000 qdisc noqueue state UP
# vlan@enp0s3 id 100
```

---

## 2. DHCP Server Configuration

### 2a. Install

```bash
# RHEL
dnf install -y dhcp-server

# Debian/Ubuntu
apt install -y isc-dhcp-server
```

### 2b. Configuration

```bash
# /etc/dhcp/dhcpd.conf
#
# PXE Boot DHCP — VLAN 100
#

authoritative;

# Bind to VLAN interface only
# (set DHCPDARGS="-cf /etc/dhcp/dhcpd.conf enp0s3.100" in /etc/default/isc-dhcp-server)

option space pxewebboot;
option pxewebboot.capabilities code 157 = string;
option pxewebboot.url code 158 = text;

subnet 10.100.0.0 netmask 255.255.255.0 {
    range 10.100.0.10 10.100.0.200;
    option routers              10.100.0.1;
    option subnet-mask          255.255.255.0;

    # MTU — DHCP Option 15 (Path MTU)
    # Clients that respect this will set their NIC MTU during boot
    option interface-mtu         9000;

    # PXE boot files
    option architecture-type    93;

    # For legacy PXE (pxelinux.0)
    class "pxeclients" {
        match if substring(option vendor-class-identifier, 0, 9) = "PXEClient";

        if option architecture-type = 00:07 {
            # UEFI x86_64
            filename "grub/x86_64-efi/grub.efi";
        } else if option architecture-type = 00:06 {
            # UEFI x86_64 (alternate)
            filename "grub/x86_64-efi/grub.efi";
        } else if option architecture-type = 00:09 {
            # UEFI ARM64
            filename "grub/arm64-efi/grubaa64.efi";
        } else {
            # Legacy BIOS
            filename "pxelinux.0";
        }
    }

    # For iPXE chainload (if client already booted iPXE)
    class "ipxe" {
        match if substring(option vendor-class-identifier, 0, 4) = "iPXE";
        # Hand off to iPXE script or direct to GRUB
        filename "http://10.100.0.1/ipxe/boot.ipxe";
    }

    next-server 10.100.0.1;
}
```

### 2c. Bind DHCP to VLAN interface only

**RHEL/CentOS:**

```bash
# /etc/sysconfig/dhcpd
DHCPDARGS="enp0s3.100"
```

**Debian/Ubuntu:**

```bash
# /etc/default/isc-dhcp-server
INTERFACESv4="enp0s3.100"
```

### 2d. Start and enable

```bash
systemctl enable --now dhcpd
# or
systemctl enable --now isc-dhcp-server
```

---

## 3. TFTP Server Configuration

### 3a. Install

```bash
# RHEL
dnf install -y tftp-server xinetd

# Debian/Ubuntu
apt install -y tftpd-hpa
```

### 3b. RHEL/CentOS (xinetd-based)

```bash
# /etc/xinetd.d/tftp
service tftp
{
    socket_type     = dgram
    protocol        = udp
    wait            = yes
    user            = root
    server          = /usr/sbin/in.tftpd
    server_args     = -s /var/lib/tftpboot
    disable         = no
    per_source      = 11
    cps             = 100 2
}
```

```bash
systemctl enable --now xinetd
```

### 3b. Debian/Ubuntu (standalone)

```bash
# /etc/default/tftpd-hpa
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS="10.100.0.1:69"
TFTP_OPTIONS="--secure"
```

```bash
systemctl enable --now tftpd-hpa
```

---

## 4. PXE Boot Files

### 4a. Install bootloaders

```bash
# RHEL
dnf install -y grub2-efi-x64 grub2-tools

# Debian/Ubuntu
apt install -y grub-efi-amd64-bin
```

### 4b. Directory structure

```
/var/lib/tftpboot/
├── pxelinux.0                    # Legacy BIOS bootloader
├── pxelinux.cfg/
│   └── default                   # PXELINUX config
├── grub/
│   ├── x86_64-efi/
│   │   ├── grub.efi              # UEFI bootloader
│   │   └── grub.cfg              # GRUB config
│   └── arm64-efi/
│       ├── grubaa64.efi
│       └── grub.cfg
├── ipxe/
│   ├── boot.ipxe                 # iPXE script (VLAN + MTU)
│   └── snponly.ipxe              # Standalone iPXE binary
├── images/
│   └── rhel9/
│       ├── initrd.img            # RHEL initramfs
│       ├── vmlinuz               # RHEL kernel
│       └── boot.iso.extracted/   # Extracted ISO contents
└── ks/
    └── rhel9.cfg                 # Kickstart file
```

### 4c. Copy PXE bootloader

```bash
# Get pxelinux.0
dnf install -y syslinux
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/

# Get iPXE standalone binary
curl -o /var/lib/tftpboot/ipxe/snponly.ipxe \
  https://github.com/ipxe/ipxe/releases/download/dist/ipxe-20240126.zip
# Extract snponly.ipxe from the zip
```

### 4d. Copy GRUB EFI files

```bash
# x86_64 UEFI
mkdir -p /var/lib/tftpboot/grub/x86_64-efi
cp /boot/efi/EFI/*/grubx64.efi /var/lib/tftpboot/grub/x86_64-efi/grub.efi
```

### 4e. Extract RHEL boot files

```bash
# Mount RHEL ISO
mkdir -p /mnt/rheliso
mount -o loop /var/iso/rhel-9.5-x86_64-dvd.iso /mnt/rheliso

# Copy kernel and initrd
mkdir -p /var/lib/tftpboot/images/rhel9
cp /mnt/rheliso/Isolinux/vmlinuz /var/lib/tftpboot/images/rhel9/
cp /mnt/rheliso/Isolinux/initrd.img /var/lib/tftpboot/images/rhel9/

umount /mnt/rheliso
```

---

## 5. Bootloader Configuration

### 5a. PXELINUX (Legacy BIOS)

```bash
# /var/lib/tftpboot/pxelinux.cfg/default
DEFAULT rhel9

TIMEOUT 60
PROMPT 0

LABEL rhel9
    KERNEL images/rhel9/vmlinuz
    APPEND initrd=images/rhel9/initrd.img \
           ip=dhcp \
           ks=http://10.100.0.1/ks/rhel9.cfg \
           ifname=mac:aa:bb:cc:dd:ee:ff \
           rd.net.timeout.carrier=10
```

### 5b. GRUB EFI (UEFI x86_64)

```bash
# /var/lib/tftpboot/grub/x86_64-efi/grub.cfg
set default=0
set timeout=5

menuentry "Install RHEL 9 (PXE)" {
    linuxefi images/rhel9/vmlinuz \
        ip=dhcp \
        ks=http://10.100.0.1/ks/rhel9.cfg \
        rd.net.timeout.carrier=10
    initrdefi images/rhel9/initrd.img
}
```

### 5c. GRUB EFI (UEFI ARM64)

```bash
# /var/lib/tftpboot/grub/arm64-efi/grub.cfg
set default=0
set timeout=5

menuentry "Install RHEL 9 (PXE ARM64)" {
    linuxefi images/rhel9/vmlinuz \
        ip=dhcp \
        ks=http://10.100.0.1/ks/rhel9.cfg \
        rd.net.timeout.carrier=10
    initrdefi images/rhel9/initrd.img
}
```

---

## 6. iPXE Configuration (Advanced — VLAN + MTU Control)

iPXE gives you client-side VLAN tagging and MTU control that plain PXE cannot do. Use this when target NIC ports are trunked or when you need to negotiate jumbo frames.

### 6a. iPXE Boot Script

```bash
# /var/lib/tftpboot/ipxe/boot.ipxe
#!ipxe

# Set VLAN tag on the client NIC
# This creates a virtual interface with the 802.1Q tag
set vlan-id 100

# Set MTU on the client NIC (jumbo frames)
set mtu-size 9000

# Configure the interface with VLAN and MTU
iflink ${net0/mac} mtu ${mtu-size}

# If the switch port is a trunk, tag the interface:
# vlan 0 ${net0} ${vlan-id}

# Set DHCP options for MTU
dhcp net0

# Chainload to RHEL kernel via HTTP
kernel http://10.100.0.1/images/rhel9/vmlinuz \
    ip=dhcp \
    ks=http://10.100.0.1/ks/rhel9.cfg \
    rd.net.timeout.carrier=10
initrd http://10.100.0.1/images/rhel9/initrd.img
boot
```

### 6b. DHCP handoff to iPXE

Add to dhcpd.conf (already included above in the `class "ipxe"` block):

```
class "ipxe" {
    match if substring(option vendor-class-identifier, 0, 4) = "iPXE";
    filename "http://10.100.0.1/ipxe/boot.ipxe";
}
```

### 6c. Force iPXE boot from PXELINUX

```bash
# /var/lib/tftpboot/pxelinux.cfg/default
DEFAULT ipxe

LABEL ipxe
    KERNEL ipxe/snponly.ipxe
    APPEND http://10.100.0.1/ipxe/boot.ipxe
```

---

## 7. HTTP Server (for Kickstart + Files)

iPXE and modern kickstart work best over HTTP, not TFTP.

### 7a. Install and configure

```bash
dnf install -y nginx
# or
apt install -y nginx
```

### 7b. Nginx config

```nginx
# /etc/nginx/conf.d/pxe.conf
server {
    listen 10.100.0.1:80;
    server_name pxe-server;

    root /var/lib/tftpboot;

    # Kickstart files
    location /ks/ {
        autoindex on;
    }

    # Boot images (for iPXE HTTP boot)
    location /images/ {
        autoindex on;
    }

    # iPXE scripts
    location /ipxe/ {
        autoindex on;
        add_header Content-Type text/plain;
    }

    # GRUB files
    location /grub/ {
        autoindex on;
    }
}
```

```bash
systemctl enable --now nginx
```

---

## 8. Kickstart Configuration

### 8a. RHEL 9 Kickstart with MTU + VLAN

```bash
# /var/lib/tftpboot/ks/rhel9.cfg
#
# RHEL 9 Kickstart — PXE with VLAN + Jumbo Frames
#

# Platform
ignoredisk --only-use=sda

# Network — VLAN interface with MTU
network --bootproto=dhcp --device=eth0 --activate --noipv6
network --bootproto=static --device=eth0.100 --vlan=id:100 \
        --ip=10.100.0.50 --netmask=255.255.255.0 \
        --gateway=10.100.0.1 --nameserver=10.100.0.1 \
        --mtu=9000 --activate

# If the installed system should use DHCP on the VLAN:
# network --bootproto=dhcp --device=eth0.100 --vlan=id:100 \
#         --mtu=9000 --activate

# Reboot after install
reboot

# Firewall
firewall --enabled --service=ssh

# Root password (use --iscrypted for production)
rootpw --plaintext changeme

# Partitioning
clearpart --all --initlabel
part /boot --fstype=xfs --size=1024
part swap --size=8192
part / --fstype=xfs --grow --size=1

# Installation source
url --url=http://10.100.0.1/rhel9/iso/

# Keyboard / language
keyboard us
lang en_US
timezone UTC --utc

# Packages
%packages
@core
@^minimal-environment
kexec-tools
%end

# Post-install: ensure MTU persists
%post
# Write NetworkManager connection with MTU
cat > /etc/NetworkManager/system-connections/vlan100.nmconnection <<'EOF'
[connection]
id=vlan100
type=vlan
interface-name=eth0.100
vlan.parent=eth0

[vlan]
id=100

[ipv4]
method=auto

[ethernet]
mtu=9000
EOF

chmod 600 /etc/NetworkManager/system-connections/vlan100.nmconnection
%end
```

### 8b. Serve RHEL ISO contents via HTTP

```bash
mkdir -p /var/lib/tftpboot/rhel9/iso
mount -o loop,ro /var/iso/rhel-9.5-x86_64-dvd.iso /var/lib/tftpboot/rhel9/iso
# Or copy contents:
# cp -a /mnt/rheliso/* /var/lib/tftpboot/rhel9/iso/
```

---

## 9. Firewall Rules

```bash
# RHEL/CentOS (firewalld)
firewall-cmd --permanent --add-port=67/udp    # DHCP server
firewall-cmd --permanent --add-port=68/udp    # DHCP client
firewall-cmd --permanent --add-port=69/udp    # TFTP
firewall-cmd --permanent --add-port=80/tcp    # HTTP (kickstart)
firewall-cmd --permanent --add-port=4011/udp  # PXE boot (optional)
firewall-cmd --reload

# Debian/Ubuntu (ufw)
ufw allow 67/udp
ufw allow 68/udp
ufw allow 69/udp
ufw allow 80/tcp
```

---

## 10. Switch Configuration

### 10a. Cisco IOS/NX-OS

```cisco
! Port connected to PXE server
interface Ethernet1/1
 description PXE-Server
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 100
 mtu 9216                          ! System MTU for jumbo frames
!
! Port connected to target RHEL host (access port)
interface Ethernet1/2
 description RHEL-Target
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
!
! If target needs VLAN tag from a trunked port (with iPXE)
interface Ethernet1/3
 description RHEL-Target-Trunk
 switchport mode trunk
 switchport trunk allowed vlan 100
!
! System-wide jumbo frame support
system mtu jumbo 9216
```

### 10b. Aruba/Cisco 9300

```cisco
interface 1/1/1
 description PXE-Server
 vlan trunk native 1
 vlan trunk allowed 100
 no shutdown

interface 1/1/2
 description RHEL-Target
 vlan access 100
 no shutdown
```

### 10c. Dell/Force10

```cisco
interface Ethernet1/1
 description PXE-Server
 switchport mode trunk
 switchport trunk allowed vlan 100
 no shutdown

interface Ethernet1/2
 description RHEL-Target
 switchport mode access
 switchport access vlan 100
 no shutdown
```

---

## 11. MTU Summary

| Layer | Setting | Where |
|-------|---------|-------|
| Server VLAN interface | `mtu 9000` | `nmcli`, `netplan`, `interfaces` |
| DHCP Option 15 | `option interface-mtu 9000` | `dhcpd.conf` |
| iPXE client | `set mtu-size 9000` | `boot.ipxe` |
| Kickstart network | `--mtu=9000` | `rhel9.cfg` |
| Post-install NM | `mtu=9000` | `%post` script |
| Switch system | `system mtu jumbo 9216` | Cisco IOS/NX-OS |

The full path must support 9000-byte frames. If any hop uses 1500, jumbo frames break silently — packets are dropped or fragmented. Verify end-to-end with:

```bash
# From target, after install:
ping -M do -s 8972 10.100.0.1
# 8972 + 28 (IP+ICMP header) = 9000. If this fails, MTU mismatch exists.
```

---

## 12. Verification Checklist

- [ ] VLAN sub-interface UP with correct MTU: `ip link show enp0s3.100`
- [ ] DHCP serving on VLAN interface: `tcpdump -i enp0s3.100 port 67`
- [ ] TFTP serving files: `tftp 10.100.0.1 -c get pxelinux.0`
- [ ] HTTP serving kickstart: `curl http://10.100.0.1/ks/rhel9.cfg`
- [ ] Switch port on correct VLAN
- [ ] Switch MTU supports jumbo frames (if using MTU > 1500)
- [ ] Target NIC port is access VLAN 100 (or trunk with iPXE)
- [ ] No firewall blocking DHCP/TFTP/HTTP to VLAN subnet
- [ ] RHEL ISO mounted or copied and accessible

---

## 13. Troubleshooting

**Client gets IP but doesn't boot:**
```bash
# Check TFTP logs
journalctl -u tftp -f
# Verify file exists
ls -la /var/lib/tftpboot/pxelinux.0
```

**DHCP offers not received:**
```bash
# Verify DHCP is bound to correct interface
ss -ulnp | grep :67
# Should show binding on 10.100.0.1
```

**iPXE VLAN not working:**
```bash
# The switch port must be a trunk for the client to send tagged frames
# If the port is access, the client doesn't need VLAN tagging —
# the switch handles it. Only use iPXE VLAN when the port is trunked.
```

**MTU mismatch:**
```bash
# Test from any host on the VLAN:
ping -M do -s 8972 <target-ip>
# If it fails, find the hop with wrong MTU
```
