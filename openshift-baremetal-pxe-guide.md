# OpenShift Container Platform 4.22 — Bare Metal PXE Installation Guide

## Reference

- Red Hat Documentation: https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/installing_on_bare_metal/index
- Architecture: x86_64
- Network plugin: OVNKubernetes (default)
- Installation method: User-provisioned infrastructure with PXE boot

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Network Architecture](#2-network-architecture)
3. [DHCP Server Configuration](#3-dhcp-server-configuration)
4. [TFTP/PXE Server Configuration](#4-tftp-pxe-server-configuration)
5. [HTTP Server for RHCOS Artifacts](#5-http-server-for-rhcos-artifacts)
6. [DNS Configuration](#6-dns-configuration)
7. [Load Balancer Configuration](#7-load-balancer-configuration)
8. [OpenShift Installation Configuration](#8-openshift-installation-configuration)
9. [Generate Manifests and Ignition Configs](#9-generate-manifests-and-ignition-configs)
10. [PXE Boot Configuration](#10-pxe-boot-configuration)
11. [MTU Configuration](#11-mtu-configuration)
12. [VLAN Configuration](#12-vlan-configuration)
13. [Network Configuration Phases](#13-network-configuration-phases)
14. [Installing RHCOS via PXE](#14-installing-rhcos-via-pxe)
15. [Running the OpenShift Installation](#15-running-the-openshift-installation)
16. [Post-Installation Tasks](#16-post-installation-tasks)
17. [Troubleshooting](#17-troubleshooting)

---

## 1. Prerequisites

### Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Bootstrap node | 4 vCPU, 16 GB RAM, 120 GB SSD | Same |
| Control plane nodes (x3) | 8 vCPU, 16 GB RAM, 120 GB SSD | 16 vCPU, 64 GB RAM, 500 GB SSD |
| Compute nodes (x2+) | 8 vCPU, 16 GB RAM, 120 GB SSD | 16 vCPU, 64 GB RAM, 500 GB SSD |

### Software Requirements

- PXE-capable network interfaces on all nodes
- DHCP server (ISC DHCP or dnsmasq)
- TFTP server (or iPXE via HTTP)
- HTTP server for serving RHCOS artifacts and Ignition configs
- DNS server with forward and reverse zones
- Load balancer (HAProxy recommended)
- OpenShift installer (`openshift-install`)
- OpenShift CLI (`oc`)
- SSH key pair for node access

### Network Requirements

| Traffic Type | Ports | Protocol |
|--------------|-------|----------|
| Kubernetes API | 6443 | TCP |
| Machine Config Server | 22623 | TCP |
| Application Ingress HTTP | 80 | TCP |
| Application Ingress HTTPS | 443 | TCP |
| PXE Boot | 67/68 | UDP |
| TFTP | 69 | UDP |
| HTTP (artifacts) | 80 | TCP |
| DNS | 53 | TCP/UDP |

### Firewall Rules

Ensure the following ports are open between cluster nodes:

- **6443/TCP**: Kubernetes API (load balancer → control plane nodes)
- **22623/TCP**: Machine Config Server (control plane nodes)
- **80,443/TCP**: Application ingress (load balancer → compute nodes)
- **1936/TCP**: Ingress health check (control plane nodes)
- **9000-9999/TCP**: OVN Kubernetes interconnect (all nodes)

---

## 2. Network Architecture

### Example Topology

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

### VLAN Tagging

If using a tagged VLAN for the provisioner/provisioning network:

- **Provisioner VLAN**: VLAN 100 (example) — used for PXE boot, API, ingress
- All nodes connect to a trunk port tagged with VLAN 100
- DHCP server must have a VLAN sub-interface (e.g., `eth0.100`)
- Switch ports connected to nodes must allow VLAN 100

---

## 3. DHCP Server Configuration

### ISC DHCP Server (`/etc/dhcp/dhcpd.conf`)

```conf
# ──────────────────────────────────────────────────────────
# ISC DHCP Server Configuration for OpenShift Bare Metal
# ──────────────────────────────────────────────────────────

# Global settings
authoritative;
default-lease-time 3600;
max-lease-time 86400;

# ──────────────────────────────────────────────────────────
# VLAN 100 — Provisioning Network
# ──────────────────────────────────────────────────────────
subnet 192.168.1.0 netmask 255.255.255.0 {
    option routers                  192.168.1.1;
    option domain-name-servers      192.168.1.2;
    option domain-name              "ocp4.example.com";

    # ── PXE Boot Options ──
    option space pxelinux;
    option pxelinux.magic           0x63538263;
    option pxelinux.configfile      "pxelinux.cfg/default";
    option pxelinux.pathprefix     "pxelinux.cfg/";

    # ── iPXE Boot (preferred for HTTP-based boot) ──
    option space iscsi;
    option iscsi.initiator          "iqn.2025-01.com.ocp4:initiator";

    # Set bootfile to iPXE chainloader
    # Option A: HTTP-based iPXE
    filename "http://192.168.1.10/pxe/boot.ipxe";

    # Option B: Traditional TFTP iPXE (if not using HTTP)
    # filename "ipxe.kpxe";

    # ── MTU Configuration ──
    # Option 26: Interface MTU
    # Set to 9000 for jumbo frames (must match switch port MTU)
    option interface-mtu 9000;

    # ── Bootstrap Node ──
    host bootstrap {
        hardware ethernet aa:bb:cc:dd:ee:01;
        fixed-address 192.168.1.96;
        option host-name "bootstrap.ocp4.example.com";
    }

    # ── Control Plane Nodes ──
    host master-1 {
        hardware ethernet aa:bb:cc:dd:ee:02;
        fixed-address 192.168.1.97;
        option host-name "master-1.ocp4.example.com";
    }

    host master-2 {
        hardware ethernet aa:bb:cc:dd:ee:03;
        fixed-address 192.168.1.98;
        option host-name "master-2.ocp4.example.com";
    }

    host master-3 {
        hardware ethernet aa:bb:cc:dd:ee:04;
        fixed-address 192.168.1.99;
        option host-name "master-3.ocp4.example.com";
    }

    # ── Compute Nodes ──
    host compute-1 {
        hardware ethernet aa:bb:cc:dd:ee:05;
        fixed-address 192.168.1.100;
        option host-name "compute-1.ocp4.example.com";
    }

    host compute-2 {
        hardware ethernet aa:bb:cc:dd:ee:06;
        fixed-address 192.168.1.101;
        option host-name "compute-2.ocp4.example.com";
    }
}

# ──────────────────────────────────────────────────────────
# PXELINUX Config Files ────────────────────────────────────
# (Only needed if using pxelinux instead of iPXE)
# ──────────────────────────────────────────────────────────
# pxelinux configuration is served from TFTP root
# See Section 4 for details
```

### dnsmasq Alternative (simpler for small deployments)

```conf
# /etc/dnsmasq.conf
# ──────────────────────────────────────────────────────────
# dnsmasq — DHCP + TFTP + DNS in one server
# ──────────────────────────────────────────────────────────

# Interfaces
interface=eth0
bind-interfaces

# DHCP range
dhcp-range=192.168.1.50,192.168.1.150,12h

# DHCP options
dhcp-option=3,192.168.1.1           # Gateway
dhcp-option=6,192.168.1.2           # DNS server
dhcp-option=15,ocp4.example.com     # Domain name
dhcp-option=26,9000                 # MTU (jumbo frames)

# PXE boot
dhcp-boot=http://192.168.1.10/pxe/boot.ipxe

# Static hosts
dhcp-host=aa:bb:cc:dd:ee:01,192.168.1.96,bootstrap.ocp4.example.com,infinite
dhcp-host=aa:bb:cc:dd:ee:02,192.168.1.97,master-1.ocp4.example.com,infinite
dhcp-host=aa:bb:cc:dd:ee:03,192.168.1.98,master-2.ocp4.example.com,infinite
dhcp-host=aa:bb:cc:dd:ee:04,192.168.1.99,master-3.ocp4.example.com,infinite
dhcp-host=aa:bb:cc:dd:ee:05,192.168.1.100,compute-1.ocp4.example.com,infinite
dhcp-host=aa:bb:cc:dd:ee:06,192.168.1.101,compute-2.ocp4.example.com,infinite

# TFTP
enable-tftp
tftp-root=/var/lib/tftpboot

# DNS
domain=ocp4.example.com
local=/ocp4.example.com/
expand-hosts
address=/#.ocp4.example.com/192.168.1.96
```

---

## 4. TFTP/PXE Server Configuration

### Directory Structure

```
/var/lib/tftpboot/
├── ipxe.kpxe                          # iPXE chainloader (optional)
├── pxelinux.0                         # PXELINUX bootloader (optional)
├── ldlinux.c32                        # Syslinux module (optional)
├── libcom32.c32                       # Syslinux module (optional)
├── libutil.c32                        # Syslinux module (optional)
├── pxelinux.cfg/
│   └── default                        # PXELINUX config (optional)
└── rhcos/
    ├── rhcos-4.22.0-x86_64-kernel     # RHCOS kernel
    ├── rhcos-4.22.0-x86_64-initramfs.img
    └── rhcos-4.22.0-x86_64-rootfs.img
```

### PXELINUX Default Config (`/var/lib/tftpboot/pxelinux.cfg/default`)

```conf
DEFAULT pxeboot
TIMEOUT 20
PROMPT 0

LABEL pxeboot
    KERNEL http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-kernel
    APPEND initrd=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-initramfs.img \
        coreos.live.rootfs_url=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-rootfs.img \
        coreos.inst.install_dev=/dev/sda \
        coreos.inst.ignition_url=http://192.168.1.10/ignition/bootstrap.ign \
        ip=dhcp \
        rd.neednet=1 \
        console=tty0 console=ttyS0
```

---

## 5. HTTP Server for RHCOS Artifacts

### Server Setup (nginx example)

```nginx
# /etc/nginx/conf.d/openshift.conf
server {
    listen 80;
    server_name 192.168.1.10;

    # ── RHCOS Artifacts ──
    location /rhcos/ {
        alias /srv/openshift/rhcos/;
        autoindex on;
        client_max_body_size 5000m;
    }

    # ── Ignition Configs ──
    location /ignition/ {
        alias /srv/openshift/ignition/;
        autoindex on;
    }

    # ── iPXE Boot Script ──
    location /pxe/ {
        alias /srv/openshift/pxe/;
    }
}
```

### Directory Structure

```
/srv/openshift/
├── rhcos/
│   ├── rhcos-4.22.0-x86_64-kernel
│   ├── rhcos-4.22.0-x86_64-initramfs.img
│   └── rhcos-4.22.0-x86_64-rootfs.img
├── ignition/
│   ├── bootstrap.ign
│   ├── master.ign
│   └── worker.ign
└── pxe/
    └── boot.ipxe
```

### Downloading RHCOS Artifacts

```bash
# Get the correct RHCOS version for your OpenShift release
openshift-install coreos print-stream-json | grep -Eo '"https.*(kernel-|initramfs.|rootfs.)\w+(\.img)?"'

# Download artifacts
mkdir -p /srv/openshift/rhcos
cd /srv/openshift/rhcos

# Example URLs (replace with actual URLs from print-stream-json)
curl -O https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/pre-release/x86_64/rhcos-4.22.0-x86_64-live-kernel-x86_64
curl -O https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/pre-release/x86_64/rhcos-4.22.0-x86_64-live-initramfs.x86_64.img
curl -O https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/pre-release/x86_64/rhcos-4.22.0-x86_64-live-rootfs.x86_64.img

# Rename to match expected names
mv rhcos-4.22.0-x86_64-live-kernel-x86_64 rhcos-4.22.0-x86_64-kernel
```

---

## 6. DNS Configuration

### Forward DNS Records

| Record Type | Name | Value | Purpose |
|-------------|------|-------|---------|
| A | `api.ocp4.example.com` | 192.168.1.5 | API load balancer |
| A | `api-int.ocp4.example.com` | 192.168.1.5 | Internal API load balancer |
| A | `*.apps.ocp4.example.com` | 192.168.1.5 | Application ingress wildcard |
| A | `bootstrap.ocp4.example.com` | 192.168.1.96 | Bootstrap node |
| A | `master-1.ocp4.example.com` | 192.168.1.97 | Control plane node 1 |
| A | `master-2.ocp4.example.com` | 192.168.1.98 | Control plane node 2 |
| A | `master-3.ocp4.example.com` | 192.168.1.99 | Control plane node 3 |
| A | `compute-1.ocp4.example.com` | 192.168.1.100 | Compute node 1 |
| A | `compute-2.ocp4.example.com` | 192.168.1.101 | Compute node 2 |

### Reverse DNS Records (PTR)

| IP Address | PTR Record | Purpose |
|------------|-----------|---------|
| 192.168.1.5 | `api.ocp4.example.com` | API load balancer |
| 192.168.1.5 | `api-int.ocp4.example.com` | Internal API |
| 192.168.1.96 | `bootstrap.ocp4.example.com` | Bootstrap node |
| 192.168.1.97 | `master-1.ocp4.example.com` | Control plane node 1 |
| 192.168.1.98 | `master-2.ocp4.example.com` | Control plane node 2 |
| 192.168.1.99 | `master-3.ocp4.example.com` | Control plane node 3 |
| 192.168.1.100 | `compute-1.ocp4.example.com` | Compute node 1 |
| 192.168.1.101 | `compute-2.ocp4.example.com` | Compute node 2 |

### BIND Configuration Example

```conf
# /etc/named.conf
options {
    directory "/var/named";
    listen-on port 53 { 192.168.1.2; };
    allow-query { any; };
    recursion yes;
};

zone "ocp4.example.com" IN {
    type master;
    file "ocp4.example.com.zone";
    allow-update { none; };
};

zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "1.168.192.in-addr.arpa.zone";
    allow-update { none; };
};
```

### Forward Zone (`ocp4.example.com.zone`)

```conf
$TTL 86400
@   IN  SOA ns1.ocp4.example.com. admin.ocp4.example.com. (
            2025010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)

@       IN  NS  ns1.ocp4.example.com.
ns1     IN  A   192.168.1.2

; API and Ingress
api             IN  A   192.168.1.5
api-int         IN  A   192.168.1.5
*.apps          IN  A   192.168.1.5

; Bootstrap
bootstrap       IN  A   192.168.1.96

; Control Plane
master-1        IN  A   192.168.1.97
master-2        IN  A   192.168.1.98
master-3        IN  A   192.168.1.99

; Compute
compute-1       IN  A   192.168.1.100
compute-2       IN  A   192.168.1.101
```

### Reverse Zone (`1.168.192.in-addr.arpa.zone`)

```conf
$TTL 86400
@   IN  SOA ns1.ocp4.example.com. admin.ocp4.example.com. (
            2025010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)

@       IN  NS  ns1.ocp4.example.com.

; API load balancer
5         IN  PTR api.ocp4.example.com.
5         IN  PTR api-int.ocp4.example.com.

; Bootstrap
96        IN  PTR bootstrap.ocp4.example.com.

; Control Plane
97        IN  PTR master-1.ocp4.example.com.
98        IN  PTR master-2.ocp4.example.com.
99        IN  PTR master-3.ocp4.example.com.

; Compute
100       IN  PTR compute-1.ocp4.example.com.
101       IN  PTR compute-2.ocp4.example.com.
```

### DNS Validation

```bash
# Forward lookups
dig +noall +answer @192.168.1.2 api.ocp4.example.com
dig +noall +answer @192.168.1.2 api-int.ocp4.example.com
dig +noall +answer @192.168.1.2 console-openshift-console.apps.ocp4.example.com
dig +noall +answer @192.168.1.2 bootstrap.ocp4.example.com
dig +noall +answer @192.168.1.2 master-1.ocp4.example.com

# Reverse lookups
dig +noall +answer @192.168.1.2 -x 192.168.1.5
dig +noall +answer @192.168.1.2 -x 192.168.1.96
dig +noall +answer @192.168.1.2 -x 192.168.1.97
```

---

## 7. Load Balancer Configuration

### HAProxy Configuration (`/etc/haproxy/haproxy.cfg`)

```conf
global
    log         127.0.0.1 local2
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    daemon

defaults
    mode                    http
    log                     global
    option                  dontlognull
    option http-server-close
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

# ── Kubernetes API (port 6443) ──
listen api-server-6443
    bind *:6443
    mode tcp
    option  httpchk GET /readyz HTTP/1.0
    option  log-health-checks
    balance roundrobin
    # Bootstrap is backup — removed after bootstrap completes
    server bootstrap bootstrap.ocp4.example.com:6443 verify none check check-ssl inter 10s fall 2 rise 3 backup
    server master-1 master-1.ocp4.example.com:6443 weight 1 verify none check check-ssl inter 10s fall 2 rise 3
    server master-2 master-2.ocp4.example.com:6443 weight 1 verify none check check-ssl inter 10s fall 2 rise 3
    server master-3 master-3.ocp4.example.com:6443 weight 1 verify none check check-ssl inter 10s fall 2 rise 3

# ── Machine Config Server (port 22623) ──
listen machine-config-server-22623
    bind *:22623
    mode tcp
    server bootstrap bootstrap.ocp4.example.com:22623 check inter 1s backup
    server master-1 master-1.ocp4.example.com:22623 check inter 1s
    server master-2 master-2.ocp4.example.com:22623 check inter 1s
    server master-3 master-3.ocp4.example.com:22623 check inter 1s

# ── Application Ingress HTTPS (port 443) ──
listen ingress-router-443
    bind *:443
    mode tcp
    balance source
    server compute-1 compute-1.ocp4.example.com:443 check inter 1s
    server compute-2 compute-2.ocp4.example.com:443 check inter 1s

# ── Application Ingress HTTP (port 80) ──
listen ingress-router-80
    bind *:80
    mode tcp
    balance source
    server compute-1 compute-1.ocp4.example.com:80 check inter 1s
    server compute-2 compute-2.ocp4.example.com:80 check inter 1s
```

### SELinux for HAProxy

```bash
# If SELinux is enforcing, allow HAProxy to bind to custom ports
setsebool -P haproxy_connect_any=1
```

### Verification

```bash
# Verify HAProxy is listening on required ports
netstat -nltupe | grep haproxy
# Expected: 0.0.0.0:6443, 0.0.0.0:22623, 0.0.0.0:443, 0.0.0.0:80
```

---

## 8. OpenShift Installation Configuration

### Download the Installer

```bash
# Download from Red Hat Hybrid Cloud Console
# https://cloud.redhat.com/openshift/downloads

# Or download a specific version
curl -L https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.22.0/openshift-install-linux.tar.gz -o openshift-install-linux.tar.gz
tar -xvf openshift-install-linux.tar.gz
chmod +x openshift-install oc

# Install oc to PATH
sudo mv oc /usr/local/bin/
```

### Create Installation Directory

```bash
mkdir -p ~/ocp4-install
cd ~/ocp4-install
```

### Generate SSH Key

```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519_ocp4
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_ocp4
```

### Create `install-config.yaml`

```yaml
apiVersion: v1
baseDomain: example.com
compute:
  - hyperthreading: Enabled
    name: worker
    replicas: 0          # Must be 0 for user-provisioned
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  networkType: OVNKubernetes
  machineNetwork:
    - cidr: 192.168.1.0/24
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: '{"auths": {"cloud.openshift.com": {"auth": "...", "email": "..."}, ...}}'
sshKey: 'ssh-ed25519 AAAA... user@host'
```

### Key Parameters Explained

| Parameter | Value | Description |
|-----------|-------|-------------|
| `baseDomain` | `example.com` | Base domain for the cluster |
| `metadata.name` | `ocp4` | Cluster name (used in DNS records) |
| `compute.replicas` | `0` | Required for user-provisioned |
| `controlPlane.replicas` | `3` | Number of control plane nodes |
| `networking.clusterNetwork` | `10.128.0.0/14` | Pod network CIDR |
| `networking.hostPrefix` | `23` | Subnet per node (/23 = 510 pods per node) |
| `networking.networkType` | `OVNKubernetes` | Default CNI plugin |
| `networking.machineNetwork` | `192.168.1.0/24` | Node network CIDR |
| `networking.serviceNetwork` | `172.30.0.0/16` | Service IP range |
| `platform.none` | `{}` | Required for bare metal |

### Get Pull Secret

Download from: https://cloud.redhat.com/openshift/downloads

```bash
# Save pull secret
cat > pull-secret.json << 'EOF'
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "YOUR_BASE64_AUTH",
      "email": "your@email.com"
    },
    "quay.io": {
      "auth": "YOUR_BASE64_AUTH",
      "email": "your@email.com"
    }
  }
}
EOF
```

---

## 9. Generate Manifests and Ignition Configs

### Generate Kubernetes Manifests

```bash
cd ~/ocp4-install
./openshift-install create manifests --dir .
```

### Verify/Modify Scheduler Config

For a 3-node cluster with no workers, enable scheduling on masters:

```bash
# Check the scheduler config
cat manifests/cluster-scheduler-02-config.yml

# For 3-node cluster (masters schedulable), ensure:
# mastersSchedulable: true
# For clusters with workers, ensure:
# mastersSchedulable: false
```

### Generate Ignition Configs

```bash
./openshift-install create ignition-configs --dir .
```

### Output Files

```
~/ocp4-install/
├── auth/
│   ├── kubeadmin-password
│   └── kubeconfig
├── bootstrap.ign          # For bootstrap node
├── master.ign             # For control plane nodes
├── worker.ign             # For compute nodes
└── metadata.json
```

### Upload Ignition Configs to HTTP Server

```bash
# Copy ignition configs to HTTP server
scp bootstrap.ign master.ign worker.ign \
    user@192.168.1.10:/srv/openshift/ignition/

# Verify they are accessible
curl http://192.168.1.10/ignition/bootstrap.ign | head -c 100
curl http://192.168.1.10/ignition/master.ign | head -c 100
curl http://192.168.1.10/ignition/worker.ign | head -c 100
```

### Get SHA512 Digests (for verification)

```bash
sha512sum bootstrap.ign
sha512sum master.ign
sha512sum worker.ign
```

---

## 10. PXE Boot Configuration

### iPXE Boot Script (`/srv/openshift/pxe/boot.ipxe`)

```ipxe
#!ipxe

# ── iPXE Boot Script for OpenShift RHCOS ──
# ── Served via HTTP from DHCP filename option ──

# Set variables
set http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-kernel
set initrd_url http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-initramfs.img
set rootfs_url http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-rootfs.img

# ── Determine node type from MAC address ──
# Bootstrap node
if eq ${net0/mac} aa:bb:cc:dd:ee:01 then
    set ignition_url http://192.168.1.10/ignition/bootstrap.ign
    goto boot
fi

# Master nodes
if eq ${net0/mac} aa:bb:cc:dd:ee:02 then
    set ignition_url http://192.168.1.10/ignition/master.ign
    goto boot
fi
if eq ${net0/mac} aa:bb:cc:dd:ee:03 then
    set ignition_url http://192.168.1.10/ignition/master.ign
    goto boot
fi
if eq ${net0/mac} aa:bb:cc:dd:ee:04 then
    set ignition_url http://192.168.1.10/ignition/master.ign
    goto boot
fi

# Worker nodes (default)
set ignition_url http://192.168.1.10/ignition/worker.ign

:boot
kernel ${kernel_url} \
    initrd=main \
    coreos.live.rootfs_url=${rootfs_url} \
    coreos.inst.install_dev=/dev/sda \
    coreos.inst.ignition_url=${ignition_url} \
    ip=dhcp \
    rd.neednet=1 \
    console=tty0 console=ttyS0

initrd --name main ${initrd_url}
boot
```

### Alternative: PXELINUX Configuration

If using PXELINUX instead of iPXE:

```conf
# /var/lib/tftpboot/pxelinux.cfg/default
DEFAULT menu
TIMEOUT 20
PROMPT 0

LABEL bootstrap
    KERNEL http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-kernel
    APPEND initrd=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-initramfs.img \
        coreos.live.rootfs_url=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-rootfs.img \
        coreos.inst.install_dev=/dev/sda \
        coreos.inst.ignition_url=http://192.168.1.10/ignition/bootstrap.ign \
        ip=dhcp rd.neednet=1 console=tty0 console=ttyS0

LABEL master
    KERNEL http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-kernel
    APPEND initrd=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-initramfs.img \
        coreos.live.rootfs_url=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-rootfs.img \
        coreos.inst.install_dev=/dev/sda \
        coreos.inst.ignition_url=http://192.168.1.10/ignition/master.ign \
        ip=dhcp rd.neednet=1 console=tty0 console=ttyS0

LABEL worker
    KERNEL http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-kernel
    APPEND initrd=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-initramfs.img \
        coreos.live.rootfs_url=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-rootfs.img \
        coreos.inst.install_dev=/dev/sda \
        coreos.inst.ignition_url=http://192.168.1.10/ignition/worker.ign \
        ip=dhcp rd.neednet=1 console=tty0 console=ttyS0
```

### UEFI GRUB Configuration (aarch64)

```conf
# /var/lib/tftpboot/grub2/grub.cfg
menuentry 'Install RHCOS - Bootstrap' {
    linux /rhcos/rhcos-4.22.0-x86_64-kernel \
        coreos.live.rootfs_url=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-rootfs.img \
        coreos.inst.install_dev=/dev/sda \
        coreos.inst.ignition_url=http://192.168.1.10/ignition/bootstrap.ign \
        ip=dhcp rd.neednet=1 console=tty0 console=ttyS0
    initrd /rhcos/rhcos-4.22.0-x86_64-initramfs.img
}

menuentry 'Install RHCOS - Master' {
    linux /rhcos/rhcos-4.22.0-x86_64-kernel \
        coreos.live.rootfs_url=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-rootfs.img \
        coreos.inst.install_dev=/dev/sda \
        coreos.inst.ignition_url=http://192.168.1.10/ignition/master.ign \
        ip=dhcp rd.neednet=1 console=tty0 console=ttyS0
    initrd /rhcos/rhcos-4.22.0-x86_64-initramfs.img
}

menuentry 'Install RHCOS - Worker' {
    linux /rhcos/rhcos-4.22.0-x86_64-kernel \
        coreos.live.rootfs_url=http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-rootfs.img \
        coreos.inst.install_dev=/dev/sda \
        coreos.inst.ignition_url=http://192.168.1.10/ignition/worker.ign \
        ip=dhcp rd.neednet=1 console=tty0 console=ttyS0
    initrd /rhcos/rhcos-4.22.0-x86_64-initramfs.img
}
```

---

## 11. MTU Configuration

### Setting MTU at Each Layer

Jumbo frames require end-to-end MTU=9000. If jumbo frames are not supported by the physical switch infrastructure, MTU must be set at these points. If physical hardware limits apply (switch port not supporting 9000, e.g.), then adjust. Here we use MTU 9000 (Jumbo). If standard is desired: set all these MTUs to 1500.

#### 1. Physical Switch Ports

```
# Example for Cisco/Nexus
interface Ethernet1/1
  mtu 9216          # System MTU (includes headers)

# Example for Arista
interface Ethernet1
  mtu 9216

# Example for Cisco IOS
interface GigabitEthernet1/0/1
  system mtu 9216
```

#### 2. DHCP Server

```conf
# ISC DHCP
option interface-mtu 9000;

# dnsmasq
dhcp-option=26,9000
```

#### 3. iPXE Boot Script

```ipxe
# Add to iPXE script before boot
set net0/mtu 9000
```

#### 4. PXE Kernel Arguments

Add to the APPEND/kernel line:

```
ip=192.168.1.X::192.168.1.1:255.255.255.0:hostname:eth0:none:off:4.4.4.4 mtu=9000
```

Or for DHCP:

```
ip=eth0:dhcp mtu=9000
```

#### 5. Post-Installation NMState Configuration

After the cluster is running, configure MTU via MachineConfig:

```yaml
# 10-mtu-config.yml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 10-mtu-master
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - contents:
            source: data:text/plain;charset=utf-8;base64,<base64_encoded_nmstate>
          mode: 0644
          overwrite: true
          path: /etc/nmstate/openshift/cluster.yml
```

Where the NMState configuration is:

```yaml
# nmstate-mtu.yml
interfaces:
  - name: enp1s0
    type: ethernet
    state: up
    mtu: 9000
    ipv4:
      enabled: true
      dhcp: true
    ipv6:
      enabled: false
  - name: br-ex
    type: ovs-bridge
    state: up
    mtu: 9000
    bridge:
      options:
        mcast-snooping-enable: true
      port:
        - name: enp1s0
        - name: br-ex
    ipv4:
      enabled: true
      dhcp: true
      auto-route-metric: 48
    ipv6:
      enabled: false
```

```bash
# Base64 encode the NMState config
base64 -w0 nmstate-mtu.yml
```

### MTU Verification

```bash
# On installed nodes
ip link show enp1s0
# Look for: mtu 9000

# Test with ping
ping -M do -s 8972 <target_ip>
# -M do: Don't fragment
# -s 8972: 8972 bytes + 28 bytes headers = 9000 bytes total
```

---

## 12. VLAN Configuration

### Tagged VLAN Setup for OpenShift Nodes

When your nodes are connected via trunk ports with tagged VLANs, you need to configure the VLAN at multiple layers.

#### 1. Physical Switch Configuration

```
# Cisco/Nexus example
interface Ethernet1/1
  switchport mode trunk
  switchport trunk allowed vlan 100
  switchport trunk native vlan 1

# Arista example
interface Ethernet1
  switchport mode trunk
  switchport trunk allowed vlan 100
```

#### 2. DHCP Server VLAN Sub-Interface

```bash
# Create VLAN sub-interface on DHCP server
ip link add link eth0 name eth0.100 type vlan id 100
ip addr add 192.168.1.10/24 dev eth0.100
ip link set eth0.100 up

# Configure ISC DHCP to listen on VLAN interface
# In dhcpd.conf:
interface eth0.100;
```

#### 3. iPXE VLAN Configuration

```ipxe
# Configure VLAN in iPXE before booting
# Create VLAN interface
ifconf net0/1 vlan 100
set net0/1/mac ${net0/mac}
set net0/1/mtu 9000

# Use VLAN interface for DHCP
dhcp net0/1

# Then boot kernel with VLAN config
kernel ${kernel_url} \
    coreos.live.rootfs_url=${rootfs_url} \
    coreos.inst.install_dev=/dev/sda \
    coreos.inst.ignition_url=${ignition_url} \
    ip=${net0/1/ip}:${net0/1/gw}:${net0/1/netmask}:hostname:eth0.${vlan}:none \
    vlan=eth0.${vlan}:eth0 \
    rd.neednet=1 \
    console=tty0 console=ttyS0
```

#### 4. PXE Kernel Arguments with VLAN

For static IP with VLAN:

```
ip=192.168.1.96::192.168.1.1:255.255.255.0:bootstrap.ocp4.example.com:eth0.100:none
vlan=eth0.100:eth0
rd.neednet=1
```

For DHCP with VLAN:

```
ip=eth0.100:dhcp
vlan=eth0.100:eth0
rd.neednet=1
```

#### 5. Post-Installation VLAN Configuration

After RHCOS boots, configure the VLAN via NMState MachineConfig:

```yaml
# 10-vlan-config.yml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 10-vlan-worker
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - contents:
            source: data:text/plain;charset=utf-8;base64,<base64_nmstate>
          mode: 0644
          overwrite: true
          path: /etc/nmstate/openshift/worker-0.yml
```

NMState configuration:

```yaml
# nmstate-vlan.yml
interfaces:
  - name: enp1s0
    type: ethernet
    state: up
    ipv4:
      enabled: false
    ipv6:
      enabled: false
  - name: enp1s0.100
    type: vlan
    state: up
    vlan:
      base-iface: enp1s0
      id: 100
    ipv4:
      enabled: true
      dhcp: true
      auto-route-metric: 48
    ipv6:
      enabled: false
```

---

## 13. Network Configuration Phases

OpenShift bare metal has three network configuration phases:

### Phase 1: PXE Boot / Live Installer

- Network configured via DHCP or kernel arguments
- Used to download Ignition config and install RHCOS
- Configuration is temporary — lost after reboot

### Phase 2: First Boot (Ignition)

- Ignition applies initial network configuration
- Uses `networkConfig` from install-config.yaml if specified
- If not specified, DHCP is used

### Phase 3: Machine Config Operator

- MCO applies MachineConfig objects
- NMState configurations are applied
- Persistent network configuration

### Specifying Network Config in install-config.yaml

```yaml
apiVersion: v1
# ... other config ...
networkConfig: |
  interfaces:
    - name: enp1s0
      type: ethernet
      state: up
      mtu: 9000
      ipv4:
        enabled: true
        dhcp: true
        auto-route-metric: 48
      ipv6:
        enabled: false
  dns-resolver:
    config:
      server:
        - 192.168.1.2
  routes:
    config:
      - dst: 0.0.0.0/0
        next-hop-address: 192.168.1.1
        next-hop-interface: enp1s0
        metric: 100
```

---

## 14. Installing RHCOS via PXE

### Pre-Installation Checklist

- [ ] DHCP server configured with static leases for all nodes
- [ ] TFTP/HTTP server serving RHCOS artifacts
- [ ] HTTP server serving Ignition configs
- [ ] iPXE boot script or PXELINUX config in place
- [ ] DNS forward and reverse records verified
- [ ] Load balancer configured and tested
- [ ] Firewall rules in place
- [ ] Switch ports configured (VLAN tagging, MTU if needed)

### Bootstrap Node PXE Boot

1. **Power on the bootstrap node**
2. **Verify PXE boot** — node should obtain IP via DHCP and chainload to iPXE
3. **Watch iPXE boot** — kernel downloads, initramfs loads, rootfs mounts
4. **Verify Ignition** — check console for Ignition applying config:

```
Ignition: ran on 2025-01-01 00:00:00 UTC (this boot)
Ignition: user-provided config was applied
```

5. **Verify installation** — coreos-installer writes to disk, system reboots

### Control Plane Nodes PXE Boot

Repeat for each master node:

1. Power on master-1, master-2, master-3
2. Each node PXE boots with its respective Ignition config
3. Verify RHCOS installation and reboot
4. Check console for Ignition application

### Compute Nodes PXE Boot

Repeat for each worker node:

1. Power on compute-1, compute-2
2. Each node PXE boots with worker.ign
3. Verify RHCOS installation and reboot

### Post-Installation Verification

```bash
# SSH to nodes (using the SSH key from install-config.yaml)
ssh core@bootstrap.ocp4.example.com
ssh core@master-1.ocp4.example.com
ssh core@master-2.ocp4.example.com
ssh core@master-3.ocp4.example.com

# Verify network configuration
ip addr show
ip link show          # Check MTU
ip route show         # Check routes

# Verify Ignition ran
journalctl -u ignition-firstboot.service
journalctl -u ignition-dispatcher.service
```

---

## 15. Running the OpenShift Installation

### Start the Installation

```bash
cd ~/ocp4-install

# Run the installer
./openshift-install install --dir . --log-level debug

# Or wait for completion
./openshift-install wait-for install-complete --dir .
```

### Monitor Installation Progress

```bash
# Watch the installation logs
./openshift-install wait-for install-complete --dir . --log-level debug

# Check bootstrap status
./openshift-install bootstrap status --dir .

# Check cluster status
./openshift-install cluster status --dir .
```

### Expected Installation Timeline

| Phase | Duration |
|-------|----------|
| Bootstrap | 15-30 minutes |
| Control plane formation | 15-30 minutes |
| Operator installation | 30-60 minutes |
| Total | 60-120 minutes |

### Access the Cluster

```bash
# After installation completes
export KUBECONFIG=~/ocp4-install/auth/kubeconfig
export BOOTSTRAP_PASSWORD=$(cat ~/ocp4-install/auth/kubeadmin-password)

# Verify cluster access
oc whoami
oc get clusterservices
oc get nodes

# Access the web console
oc get console
```

---

## 16. Post-Installation Tasks

### Remove Bootstrap from Load Balancer

After bootstrap completes, remove bootstrap entries from HAProxy:

```bash
# Edit /etc/haproxy/haproxy.cfg
# Remove bootstrap from api-server-6443 and machine-config-server-22623
# Reload HAProxy
systemctl reload haproxy
```

### Deploy Compute Nodes

If you set `compute.replicas: 0` and want to add workers later:

```bash
# Scale worker machinesets
oc get machinesets -n openshift-machine-api
oc scale machineset <worker-machineset> --replicas=2 -n openshift-machine-api

# Or manually PXE boot additional nodes with worker.ign
```

### Apply Additional MachineConfigs

```bash
# Apply MTU configuration
oc apply -f 10-mtu-config.yml

# Apply VLAN configuration
oc apply -f 10-vlan-config.yml

# Apply br-ex customization
oc apply -f 10-br-ex-config.yml
```

### Verify Cluster Health

```bash
# Check cluster operators
oc get clusteroperators

# Check nodes
oc get nodes -o wide

# Check pods in openshift namespaces
oc get pods -n openshift-etcd
oc get pods -n openshift-apiserver
oc get pods -n openshift-ingress-operator

# Check DNS
oc get pods -n openshift-dns

# Check network
oc get network.config/cluster -o yaml
```

### Save Installation Assets

```bash
# IMPORTANT: Keep these files for cluster deletion
# - install-config.yaml
# - openshift-install binary
# - auth/kubeconfig
# - auth/kubeadmin-password
# - All manifests and Ignition configs
```

---

## 17. Troubleshooting

### PXE Boot Fails

```bash
# Check DHCP server logs
tail -f /var/log/messages | grep dhcpd

# Check TFTP server logs
tail -f /var/log/messages | grep in.tftpd

# Check HTTP server logs
tail -f /var/log/nginx/access.log

# Verify files are accessible
curl http://192.168.1.10/rhcos/rhcos-4.22.0-x86_64-kernel
curl http://192.168.1.10/ignition/bootstrap.ign
```

### Ignition Config Not Applied

```bash
# Check Ignition logs
journalctl -u ignition-firstboot.service
journalctl -u ignition-dispatcher.service

# Verify Ignition URL is correct
# Check iPXE/PXE config for correct ignition_url

# Verify HTTP server serves the file
curl -v http://192.168.1.10/ignition/bootstrap.ign
```

### Network Configuration Issues

```bash
# Check interface status
ip link show
ip addr show

# Check MTU
ip link show enp1s0
# Should show: mtu 9000

# Check routes
ip route show

# Check DNS resolution
nslookup api.ocp4.example.com
nslookup master-1.ocp4.example.com

# Check VLAN configuration
ip link show eth0.100
```

### Load Balancer Issues

```bash
# Check HAProxy status
systemctl status haproxy
haproxy -c -f /etc/haproxy/haproxy.cfg

# Check HAProxy logs
tail -f /var/log/haproxy.log

# Test API connectivity
curl -k https://api.ocp4.example.com:6443/readyz

# Test from nodes
ssh core@master-1.ocp4.example.com
curl -k https://localhost:6443/readyz
```

### Cluster Installation Stuck

```bash
# Check bootstrap status
./openshift-install bootstrap status --dir .

# Get bootstrap logs
./openshift-install gather bootstrap --name ocp4

# Check cluster operators
oc get clusteroperators

# Check specific operator logs
oc logs -n openshift-apiserver deployment/apiserver
oc logs -n openshift-etcd etcd-master-1
```

### Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| `Ignition config not found` | Wrong URL or HTTP server down | Verify URL, check HTTP server |
| `PXE boot timeout` | DHCP not responding | Check DHCP server, network connectivity |
| `Kernel panic` | Wrong kernel/initramfs | Download correct RHCOS version |
| `API connection refused` | Bootstrap not ready, LB misconfig | Check HAProxy config, wait longer |
| `Node not joining cluster` | DNS mismatch, wrong hostname | Verify DNS forward/reverse |
| `MTU mismatch` | Packet fragmentation | Ensure MTU is 9000 at all layers |

---

## Appendix A: Complete iPXE Script with VLAN and MTU

```ipxe
#!ipxe

# ── Complete iPXE Boot Script ──
# ── With VLAN tagging and jumbo frame support ──

# Set HTTP server
set server 192.168.1.10
set rhcos_ver 4.22.0
set arch x86_64

# ── VLAN Configuration ──
# Uncomment if using tagged VLAN
# set vlan_id 100
# ifconf net0/1 vlan ${vlan_id}
# set net0/1/mac ${net0/mac}
# dhcp net0/1

# ── MTU Configuration ──
set net0/mtu 9000

# ── Get IP via DHCP ──
dhcp

# ── Set RHCOS artifact URLs ──
set kernel_url http://${server}/rhcos/rhcos-${rhcos_ver}-${arch}-kernel
set initrd_url http://${server}/rhcos/rhcos-${rhcos_ver}-${arch}-initramfs.img
set rootfs_url http://${server}/rhcos/rhcos-${rhcos_ver}-${arch}-rootfs.img

# ── Determine node type from MAC address ──
# Bootstrap
if eq ${net0/mac} aa:bb:cc:dd:ee:01 then
    set ignition_url http://${server}/ignition/bootstrap.ign
    goto boot
fi

# Masters
if eq ${net0/mac} aa:bb:cc:dd:ee:02 then
    set ignition_url http://${server}/ignition/master.ign
    goto boot
fi
if eq ${net0/mac} aa:bb:cc:dd:ee:03 then
    set ignition_url http://${server}/ignition/master.ign
    goto boot
fi
if eq ${net0/mac} aa:bb:cc:dd:ee:04 then
    set ignition_url http://${server}/ignition/master.ign
    goto boot
fi

# Workers (default)
set ignition_url http://${server}/ignition/worker.ign

:boot
kernel ${kernel_url} \
    initrd=main \
    coreos.live.rootfs_url=${rootfs_url} \
    coreos.inst.install_dev=/dev/sda \
    coreos.inst.ignition_url=${ignition_url} \
    ip=dhcp \
    rd.neednet=1 \
    console=tty0 console=ttyS0

initrd --name main ${initrd_url}
boot
```

## Appendix B: Complete install-config.yaml

```yaml
apiVersion: v1
baseDomain: example.com
compute:
  - hyperthreading: Enabled
    name: worker
    replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  networkType: OVNKubernetes
  machineNetwork:
    - cidr: 192.168.1.0/24
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: '{"auths": {"cloud.openshift.com": {"auth": "...", "email": "..."}}}'
sshKey: 'ssh-ed25519 AAAA... user@host'
networkConfig: |
  interfaces:
    - name: enp1s0
      type: ethernet
      state: up
      mtu: 9000
      ipv4:
        enabled: true
        dhcp: true
        auto-route-metric: 48
      ipv6:
        enabled: false
  dns-resolver:
    config:
      server:
        - 192.168.1.2
  routes:
    config:
      - dst: 0.0.0.0/0
        next-hop-address: 192.168.1.1
        next-hop-interface: enp1s0
        metric: 100
```

## Appendix C: Quick Reference — File Locations

| File | Location | Purpose |
|------|----------|---------|
| DHCP config | `/etc/dhcp/dhcpd.conf` | IP assignment, PXE boot |
| TFTP root | `/var/lib/tftpboot/` | PXE boot files |
| HTTP root | `/srv/openshift/` | RHCOS + Ignition + iPXE |
| RHCOS artifacts | `/srv/openshift/rhcos/` | Kernel, initramfs, rootfs |
| Ignition configs | `/srv/openshift/ignition/` | bootstrap.ign, master.ign, worker.ign |
| iPXE script | `/srv/openshift/pxe/boot.ipxe` | Network boot script |
| HAProxy config | `/etc/haproxy/haproxy.cfg` | Load balancing |
| BIND zones | `/var/named/` | DNS forward/reverse |
| install-config | `~/ocp4-install/` | OpenShift installation config |
| Kubeconfig | `~/ocp4-install/auth/kubeconfig` | Cluster access |
