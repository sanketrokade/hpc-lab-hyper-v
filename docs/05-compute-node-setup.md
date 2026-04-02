# Phase 5: Compute Node Setup

## Overview
Deploy two compute nodes, configure cluster networking, and mount shared
storage from the headnode via NFS.

## Prerequisites
- Headnode fully configured (Phase 4 complete)
- Hyper-V switches (LabSwitch1, ClusterSwitch) exist
- Headnode NFS exports active on 10.10.10.0/24
- Rocky 9.7 minimal ISO available

## 5.1 VM Creation (PowerShell as Administrator)

### Clean Up and Create VMs

```powershell
# Remove old empty folders if they exist
Remove-Item "D:\sanket\VMs\cluster\compute1" -Recurse -Force
Remove-Item "D:\sanket\VMs\cluster\compute2" -Recurse -Force

# ===== COMPUTE-1 =====
New-VM -Name "compute-1" `
  -MemoryStartupBytes 4GB `
  -Generation 2 `
  -NewVHDPath "D:\sanket\VMs\cluster\compute-1\compute1-os.vhdx" `
  -NewVHDSizeBytes 20GB `
  -SwitchName "ClusterSwitch" `
  -Path "D:\sanket\VMs\cluster"

Set-VM -Name "compute-1" -ProcessorCount 4 -StaticMemory -CheckpointType Disabled
Set-VMFirmware -VMName "compute-1" -EnableSecureBoot Off
Add-VMDvdDrive -VMName "compute-1" -Path "D:\sanket\softwares\iso\Rocky-9.7-x86_64-minimal.iso"
$dvd1 = Get-VMDvdDrive -VMName "compute-1"
Set-VMFirmware -VMName "compute-1" -FirstBootDevice $dvd1

# ===== COMPUTE-2 =====
New-VM -Name "compute-2" `
  -MemoryStartupBytes 4GB `
  -Generation 2 `
  -NewVHDPath "D:\sanket\VMs\cluster\compute-2\compute2-os.vhdx" `
  -NewVHDSizeBytes 20GB `
  -SwitchName "ClusterSwitch" `
  -Path "D:\sanket\VMs\cluster"

Set-VM -Name "compute-2" -ProcessorCount 4 -StaticMemory -CheckpointType Disabled
Set-VMFirmware -VMName "compute-2" -EnableSecureBoot Off
Add-VMDvdDrive -VMName "compute-2" -Path "D:\sanket\softwares\iso\Rocky-9.7-x86_64-minimal.iso"
$dvd2 = Get-VMDvdDrive -VMName "compute-2"
Set-VMFirmware -VMName "compute-2" -FirstBootDevice $dvd2
```

### Verify VM Configuration

```powershell
Get-VM compute-1, compute-2 | Format-Table Name, State, MemoryStartup, ProcessorCount, Generation
Get-VMNetworkAdapter -VMName compute-1, compute-2 | Format-Table VMName, SwitchName
Get-VMHardDiskDrive -VMName compute-1, compute-2 | Format-Table VMName, Path
Get-VMDvdDrive -VMName compute-1, compute-2 | Format-Table VMName, Path
```

### Expected Output

| Property    | compute-1                         | compute-2                         |
|-------------|-----------------------------------|-----------------------------------|
| Memory      | 4GB (static)                      | 4GB (static)                      |
| CPUs        | 4                                 | 4                                 |
| Generation  | 2                                 | 2                                 |
| Switch      | ClusterSwitch                     | ClusterSwitch                     |
| OS Disk     | compute-1\compute1-os.vhdx (20GB) | compute-2\compute2-os.vhdx (20GB) |
| Checkpoints | Disabled                          | Disabled                          |
| Secure Boot | Off                               | Off                               |

### Key Design Decisions
- **ClusterSwitch only** — compute nodes have no direct internet access, all traffic routes through headnode
- **No data disk** — shared storage comes from headnode via NFS
- **Static memory** — HPC needs predictable resource allocation
- **Checkpoints disabled** — not needed for lab, avoids disk bloat
- **Secure Boot off** — Rocky Linux boot issues with Gen 2 Secure Boot

## 5.2 OS Installation

### Start and Connect

```powershell
Start-VM -Name "compute-1"
vmconnect localhost compute-1
```

### Installation Checklist (Same for Both Nodes)

| Setting                  | Value                                    |
|--------------------------|------------------------------------------|
| Keyboard                 | English (US)                             |
| Time & Date              | Asia/Kolkata                             |
| Software Selection       | Minimal Install                          |
| Installation Destination | 20GB disk, Custom, Standard Partition    |
| Root Password            | Set, SSH access enabled                  |
| User Creation            | None (managed centrally later)           |

### Partitioning — Standard Partitions (NOT LVM)

| Mount Point | Size      | Filesystem           |
|-------------|-----------|----------------------|
| /boot/efi   | 600 MiB   | EFI System Partition |
| /boot       | 1 GiB     | xfs                  |
| swap        | 2 GiB     | swap                 |
| /           | Remaining | xfs                  |

### Hostname
- compute-1: `compute-1` (dash, not letter L)
- compute-2: `compute-2`

### Common Mistakes to Avoid
- Using Automatic partitioning (creates LVM)
- Forgetting to change partitioning scheme dropdown to Standard Partition
- Typo in hostname (compute1 vs compute-1)
- Configuring NIC during install (do it post-install)

### Post-Install Verification

```bash
hostnamectl
lsblk -f
cat /etc/fstab
free -h
```

### Expected Verification Output
- Hostname matches (compute-1 or compute-2)
- Four standard partitions: sda1 (vfat), sda2 (xfs), sda3 (swap), sda4 (xfs)
- No LVM, no volume groups
- All mounts by UUID in fstab
- ~3.6GB total memory shown

## 5.3 Network Configuration

### Compute Node Network Design
- Single interface: eth0 on ClusterSwitch (private)
- Static IP, gateway points to headnode (10.10.10.1)
- DNS through headnode's masquerade to internet
- No direct internet access

### Configure Static IP

**On compute-1:**

```bash
nmcli con add con-name "cluster" \
  ifname eth0 \
  type ethernet \
  ipv4.addresses 10.10.10.11/24 \
  ipv4.gateway 10.10.10.1 \
  ipv4.dns 8.8.8.8 \
  ipv4.method manual \
  autoconnect yes

nmcli con up cluster
```

**On compute-2:**

```bash
nmcli con add con-name "cluster" \
  ifname eth0 \
  type ethernet \
  ipv4.addresses 10.10.10.12/24 \
  ipv4.gateway 10.10.10.1 \
  ipv4.dns 8.8.8.8 \
  ipv4.method manual \
  autoconnect yes

nmcli con up cluster
```

### Verify Networking

```bash
ip a show eth0
nmcli con show cluster | grep -E "ipv4\.(addresses|gateway|dns|method)"
```

### Test Connectivity (Three Layers)

```bash
# Layer 1: Cluster connectivity
ping -c 2 10.10.10.1

# Layer 2: Internet routing through headnode masquerade
ping -c 2 8.8.8.8

# Layer 3: DNS resolution
ping -c 2 google.com
```

All three must succeed. If layer 1 fails, check IP/switch. If layer 2 fails,
check headnode ip_forward and masquerade. If layer 3 fails, check DNS setting.

## 5.4 Host Name Resolution

### Add /etc/hosts on All Three Nodes

```bash
cat >> /etc/hosts << 'EOF'
10.10.10.1   headnode
10.10.10.11  compute-1
10.10.10.12  compute-2
EOF
```

### Verify

```bash
cat /etc/hosts
ping -c 1 headnode
ping -c 1 compute-1
ping -c 1 compute-2
```

**Why /etc/hosts instead of DNS?**
Three nodes don't justify running a DNS server. /etc/hosts is simple, reliable,
and has zero dependencies. In larger clusters (50+ nodes), you'd switch to DNS
or let a provisioning system manage it.

## 5.5 NFS Client Mounts

### Install NFS Utilities

```bash
dnf install -y nfs-utils
```

### Verify Exports Are Visible

```bash
showmount -e headnode
```

Expected output:

```
Export list for headnode:
/export/scratch 10.10.10.0/24
/export/apps    10.10.10.0/24
/export/home    10.10.10.0/24
```

**Note:** `showmount -e` is a client-side tool that queries a remote NFS server's
export list. It works across the network — you don't need to be on the server.

### Create Mount Points and Add to fstab

```bash
mkdir -p /export/home /export/apps /export/scratch

cat >> /etc/fstab << 'EOF'
headnode:/export/home    /export/home    nfs defaults,_netdev 0 0
headnode:/export/apps    /export/apps    nfs defaults,_netdev 0 0
headnode:/export/scratch /export/scratch nfs defaults,_netdev 0 0
EOF

mount -a
```

### Why _netdev?

During boot, Linux mounts filesystems from /etc/fstab before the network is
fully up. NFS shares need the network. Without `_netdev`, the system tries to
mount network shares with no network available — mount fails, boot hangs or
drops to emergency mode.

`_netdev` tells systemd: "This filesystem lives on the network. Don't attempt
mounting until network-online.target is reached."

In production, a missing `_netdev` on NFS mounts has caused entire racks of
compute nodes to hang on reboot.

### Verify Mounts

```bash
df -h | grep export
```

Expected output:

```
headnode:/export/home      10G  103M  9.9G   2% /export/home
headnode:/export/apps      20G  174M   20G   1% /export/apps
headnode:/export/scratch   15G  138M   15G   1% /export/scratch
```

### Test Cross-Node Shared Storage

From compute-1:

```bash
touch /export/scratch/test-from-compute1
```

From compute-2:

```bash
ls -l /export/scratch/
```

If compute-2 sees the file compute-1 created, shared storage is working
across the cluster. Clean up after testing:

```bash
rm /export/scratch/test-from-compute1
```

## Final Cluster State After Phase 5

| Node      | IP          | Gateway    | NFS Mounts | Internet     |
|-----------|-------------|------------|------------|--------------|
| headnode  | 10.10.10.1  | —          | Server     | Direct       |
| compute-1 | 10.10.10.11 | 10.10.10.1 | Client     | Via headnode |
| compute-2 | 10.10.10.12 | 10.10.10.1 | Client     | Via headnode |

All nodes can resolve each other by hostname. Compute nodes access
internet through headnode's IP masquerade. Shared storage is accessible
on all nodes at /export/{home,apps,scratch}.
