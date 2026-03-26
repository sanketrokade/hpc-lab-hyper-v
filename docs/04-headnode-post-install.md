# Phase 4: Head Node Post-Install Configuration

## Overview
Configuring networking, firewall, gateway, LVM storage, and NFS server on the headnode.
This phase transforms a bare Rocky Linux install into a functioning cluster head node.

## Network Configuration

### Identifying Interfaces
Hyper-V NICs use eth naming. Cross-reference MAC addresses between VM and host
to determine which interface connects to which switch.

```bash
# Inside VM
ip a
# Shows eth0 MAC and eth1 MAC

# From host PowerShell
Get-VMNetworkAdapter -VMName "headnode" | Format-Table Name, SwitchName, MacAddress -AutoSize
```

**Mapping result:**
- eth0 → LabSwitch1 (internet via NAT)
- eth1 → ClusterSwitch (cluster internal)

### eth0 — LabSwitch1 (Internet)
```bash
sudo nmcli con mod eth0 ipv4.addresses 192.168.30.2/28
sudo nmcli con mod eth0 ipv4.gateway 192.168.30.1
sudo nmcli con mod eth0 ipv4.dns 8.8.8.8
sudo nmcli con mod eth0 ipv4.method manual
sudo nmcli con mod eth0 connection.zone public
sudo nmcli con up eth0
```

### eth1 — ClusterSwitch (Cluster)
```bash
sudo nmcli con mod eth1 ipv4.addresses 10.10.10.1/24
sudo nmcli con mod eth1 ipv4.method manual
sudo nmcli con mod eth1 connection.zone trusted
sudo nmcli con up eth1
```

**Why no gateway on eth1?** A system can only have one default gateway.
Two default routes cause unpredictable packet paths. Gateway belongs on eth0
(path to internet). Cluster traffic stays local — no gateway needed.

**Why no DNS on eth1?** Compute nodes resolve hostnames via /etc/hosts,
not DNS. No need for DNS on the cluster network.

### Verification
```bash
ip a
# eth0: 192.168.30.2/28
# eth1: 10.10.10.1/24

ip route
# default via 192.168.30.1 dev eth0
# 10.10.10.0/24 dev eth1
# 192.168.30.0/28 dev eth0

ping -c 3 8.8.8.8
# Confirms internet connectivity

ping -c 3 google.com
# Confirms DNS resolution
```

## Firewall Zones

### Why Separate Zones?
Each interface has a different trust level:
- **eth0 (public):** Faces the NAT network. Restrict access.
- **eth1 (trusted):** Cluster-internal only. Allow everything between nodes.

### Verification
```bash
sudo firewall-cmd --get-active-zones
# public: eth0
# trusted: eth1
```

## /etc/hosts
```bash
sudo tee /etc/hosts << 'EOF'
127.0.0.1   localhost localhost.localdomain
::1         localhost localhost.localdomain

# Cluster nodes
10.10.10.1   headnode
10.10.10.11  compute-1
10.10.10.12  compute-2
EOF
```

**Why /etc/hosts instead of DNS?** For a 3-node lab cluster, a full DNS server
(BIND or dnsmasq) is overhead. /etc/hosts is simple, reliable, and what many
small HPC clusters actually use. DNS comes later if we scale up.

### Verification
```bash
ping -c 1 headnode
# PING headnode (10.10.10.1) — resolves correctly
```

## Gateway Setup

### Purpose
Compute nodes are on ClusterSwitch (Private) — no internet access by design.
But they need internet for package installation and updates. The headnode acts
as a gateway: compute nodes send traffic to 10.10.10.1, headnode forwards it
through eth0 to the NAT network, and out to the internet.

### Enable IP Forwarding
```bash
sudo tee /etc/sysctl.d/99-cluster.conf << 'EOF'
net.ipv4.ip_forward = 1
EOF
sudo sysctl -p /etc/sysctl.d/99-cluster.conf
```

**Why a drop-in file?** Editing /etc/sysctl.conf directly risks being overwritten
by package updates. Drop-in files in /etc/sysctl.d/ survive updates.

### Enable Masquerading
```bash
sudo firewall-cmd --zone=public --add-masquerade --permanent
sudo firewall-cmd --reload
```

**What is masquerading?** NAT for outgoing traffic. When compute-1 sends a packet
to 8.8.8.8, the headnode rewrites the source IP from 10.10.10.11 to 192.168.30.2
so the reply knows how to get back. Without masquerading, replies to 10.10.10.11
would be dropped — that IP doesn't exist on the internet.

### Verification
```bash
cat /proc/sys/net/ipv4/ip_forward
# 1

sudo firewall-cmd --zone=public --query-masquerade
# yes
```

## LVM Storage (50GB Data Disk)

### Why LVM for Data?
Shared storage needs to grow. When /export/apps fills up with scientific software,
LVM lets us extend the logical volume and filesystem without downtime. The 5GB
free space in the VG is reserved for exactly this purpose.

### Create Physical Volume, Volume Group, and Logical Volumes
```bash
sudo pvcreate /dev/sdb
sudo vgcreate vg_shared /dev/sdb
sudo lvcreate -L 10G -n lv_home vg_shared
sudo lvcreate -L 20G -n lv_apps vg_shared
sudo lvcreate -L 15G -n lv_scratch vg_shared
```

### LV Layout

| LV         | Size | Mount           | FS  | Purpose                       |
|------------|------|-----------------|-----|-------------------------------|
| lv_home    | 10GB | /export/home    | xfs | User home directories         |
| lv_apps    | 20GB | /export/apps    | xfs | Compilers, MPI, software      |
| lv_scratch | 15GB | /export/scratch | xfs | Temporary job data            |
| Free       | 5GB  | —               | —   | Reserved for future expansion |

**Why apps gets 20GB?** Compilers (GCC, Intel oneAPI), MPI libraries (OpenMPI, MPICH),
scientific software (GROMACS, LAMMPS), Python environments — all installed once on
headnode and shared read-only to compute nodes. It accumulates fast.

### Format Filesystems
```bash
sudo mkfs.xfs /dev/vg_shared/lv_home
sudo mkfs.xfs /dev/vg_shared/lv_apps
sudo mkfs.xfs /dev/vg_shared/lv_scratch
```

### Create Mount Points
```bash
sudo mkdir -p /export/{home,apps,scratch}
```

**Why /export?** Convention that signals "this directory is NFS-exported."
Anyone reading the filesystem layout immediately knows these are shared.

### Add to /etc/fstab (UUID-based)
```bash
sudo blkid /dev/vg_shared/lv_home -s UUID -o value | \
  xargs -I {} echo "UUID={}  /export/home     xfs  defaults  0 0" | sudo tee -a /etc/fstab
sudo blkid /dev/vg_shared/lv_apps -s UUID -o value | \
  xargs -I {} echo "UUID={}  /export/apps     xfs  defaults  0 0" | sudo tee -a /etc/fstab
sudo blkid /dev/vg_shared/lv_scratch -s UUID -o value | \
  xargs -I {} echo "UUID={}  /export/scratch  xfs  defaults  0 0" | sudo tee -a /etc/fstab
```

### Mount and Verify
```bash
sudo mount -a
sudo systemctl daemon-reload

df -h /export/home /export/apps /export/scratch
# /dev/mapper/vg_shared-lv_home      10G  104M  9.9G   2% /export/home
# /dev/mapper/vg_shared-lv_apps      20G  175M   20G   1% /export/apps
# /dev/mapper/vg_shared-lv_scratch   15G  139M   15G   1% /export/scratch

sudo vgdisplay vg_shared | grep -E "VG Size|Free"
# VG Size  <50.00 GiB
# Free     1279 / <5.00 GiB
```

## NFS Server Setup

### Install and Enable
```bash
sudo dnf install nfs-utils -y
sudo systemctl enable --now nfs-server
```

### Configure Exports
```bash
sudo tee /etc/exports << 'EOF'
/export/home    10.10.10.0/24(rw,sync,no_root_squash)
/export/apps    10.10.10.0/24(ro,sync)
/export/scratch 10.10.10.0/24(rw,sync,no_root_squash)
EOF
sudo exportfs -rav
```

### Export Options Explained

| Directory | Access | no_root_squash | Why |
|-----------|--------|----------------|-----|
| /export/home | rw | yes | Users need to read/write their home directories |
| /export/apps | ro | no (default squash) | Software installed on headnode only, shared read-only |
| /export/scratch | rw | yes | Jobs write temporary data during computation |

**Why no_root_squash on home and scratch?** HPC jobs often run operations that
need root-level file access (creating files with specific ownership, etc.).
Root squash would map root to nobody, breaking these operations.

**Why apps is read-only?** Software is installed and compiled on the headnode.
Compute nodes only need to execute it. Read-only prevents accidental modification.

**Why 10.10.10.0/24 only?** NFS should never be exposed to the NAT network.
Even if the headnode is compromised via eth0, the attacker would need to pivot
to a completely separate network (ClusterSwitch) to reach NFS. Defense in depth.

### Open Firewall for NFS
```bash
sudo firewall-cmd --zone=trusted --add-service=nfs --permanent
sudo firewall-cmd --zone=trusted --add-service=mountd --permanent
sudo firewall-cmd --zone=trusted --add-service=rpc-bind --permanent
sudo firewall-cmd --reload
```

**Why three services?** NFS depends on a protocol stack:
- **rpc-bind:** Maps RPC program numbers to ports. NFS client asks "where is NFS?" and rpc-bind answers.
- **mountd:** Handles mount requests. Client says "I want to mount /export/home" and mountd checks /etc/exports.
- **nfs:** The actual file serving protocol. Handles read/write operations after mount is established.

### Verification
```bash
sudo exportfs -v
# /export/home    10.10.10.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
# /export/apps    10.10.10.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,ro,secure,root_squash,no_all_squash)
# /export/scratch 10.10.10.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)

sudo firewall-cmd --zone=trusted --list-services
# mountd nfs rpc-bind
```

## Summary of Headnode State After This Phase

| Component | Status |
|-----------|--------|
| eth0 (LabSwitch1) | 192.168.30.2/28, gateway 192.168.30.1, public zone |
| eth1 (ClusterSwitch) | 10.10.10.1/24, no gateway, trusted zone |
| Internet | Working (ping 8.8.8.8 and google.com) |
| IP forwarding | Enabled |
| Masquerading | Enabled on public zone |
| /etc/hosts | headnode, compute-1, compute-2 mapped |
| LVM (vg_shared) | 3 LVs (10+20+15=45GB used, 5GB free) |
| NFS exports | home(rw), apps(ro), scratch(rw) on 10.10.10.0/24 |
| Firewall (trusted) | nfs, mountd, rpc-bind allowed |
