# HPC Lab Cluster — Conversation State
# Last Updated: Phase 5 Complete (Compute Nodes Deployed)

## Who Am I Working With
- Name: Sanket
- Goal: Become an HPC System Administrator — not average, the best
- Mindset: Practical, evidence-based, wants to understand WHY not just HOW
- Skill Level: Comfortable with Linux (RHEL and Debian), used VMware and Hyper-V
- Previous HPC lab experience on VMware (completed through Phase 5)
- Rebuilding from scratch on Hyper-V for deeper learning

## Rules
1. Keep it conversational — no info dumps
2. After each milestone, provide raw documentation files for GitHub (one file per message)
3. Maintain this state file — update after each phase
4. Explain the WHY and history behind every technology choice
5. Challenge Sanket's thinking, don't just give answers
6. All VM operations via PowerShell CLI — not GUI
7. Documentation must be properly formatted markdown, copy-paste ready

## Lab Environment
- Host: HP Z8 Workstation, Windows 11
- RAM: 65GB
- CPUs: 48 logical (24 cores)
- GPU: NVIDIA RTX Ada 6000
- Hypervisor: Hyper-V
- OS: Rocky Linux 9.7 Minimal
- VM Generation: Gen 2 (UEFI)
- Repo: GitHub (hpc-lab-hyper-v)

## Cluster Architecture

| Node      | RAM | CPUs | Disks              | Network                    |
|-----------|-----|------|--------------------|----------------------------|
| headnode  | 8GB | 4    | 20GB OS + 50GB Data| LabSwitch1 + ClusterSwitch |
| compute-1 | 4GB | 4    | 20GB OS            | ClusterSwitch only         |
| compute-2 | 4GB | 4    | 20GB OS            | ClusterSwitch only         |

Total used: 16GB RAM, 12 CPUs — leaves 49GB and 36 CPUs for host.

## Hyper-V Virtual Switches

| Switch         | Type     | Purpose                     |
|----------------|----------|-----------------------------|
| vSwitch1       | External | Host internet (not used by VMs) |
| LabSwitch1     | Internal | Headnode internet via NAT   |
| ClusterSwitch  | Private  | Cluster internal only       |
| Default Switch | Internal | Hyper-V default (ignored)   |

## Host-Side NAT Configuration
- LabSwitch1 host IP: 192.168.30.1/28
- NetNat name: LabNAT
- NAT prefix: 192.168.30.0/28

## Network Design

| Network       | Subnet           | Purpose              |
|---------------|-------------------|----------------------|
| LabSwitch1    | 192.168.30.0/28   | Internet (head only) |
| ClusterSwitch | 10.10.10.0/24     | Cluster internal     |

### Static IPs
- headnode LabSwitch1: 192.168.30.2/28 (gateway: 192.168.30.1)
- headnode ClusterSwitch: 10.10.10.1/24
- compute-1 ClusterSwitch: 10.10.10.11/24 (gateway: 10.10.10.1)
- compute-2 ClusterSwitch: 10.10.10.12/24 (gateway: 10.10.10.1)

### Interfaces on headnode
- eth0: LabSwitch1, Static (192.168.30.2), MAC 00:15:5d:59:08:03, firewall public zone
- eth1: ClusterSwitch, Static (10.10.10.1), MAC 00:15:5d:59:08:04, firewall trusted zone

### Interfaces on compute-1
- eth0: ClusterSwitch, Static (10.10.10.11), MAC 00:15:5d:59:08:08
- Connection profile name: cluster

### Interfaces on compute-2
- eth0: ClusterSwitch, Static (10.10.10.12), MAC 00:15:5d:59:08:09
- Connection profile name: cluster

### Gateway Configuration
- ip_forward = 1 (set in /etc/sysctl.d/99-cluster.conf on headnode)
- Masquerading enabled on public zone
- Compute nodes use 10.10.10.1 as default gateway
- DNS: 8.8.8.8 on all nodes

## Storage Layout (headnode)

### OS Disk (20GB) — Standard Partitions, GPT/UEFI
| Partition | Mount     | Size    | FS                   |
|-----------|-----------|---------|----------------------|
| sda1      | /boot/efi | 600MB   | EFI System Partition |
| sda2      | /boot     | 1GB     | xfs                  |
| sda3      | swap      | 2GB     | swap                 |
| sda4      | /         | ~16.4GB | xfs                  |

### Data Disk (50GB) — LVM
- PV: /dev/sdb
- VG: vg_shared (50GB total, 5GB free)

| LV         | Size | Mount           | FS  |
|------------|------|-----------------|-----|
| lv_home    | 10GB | /export/home    | xfs |
| lv_apps    | 20GB | /export/apps    | xfs |
| lv_scratch | 15GB | /export/scratch | xfs |

All mounted via UUID in /etc/fstab. Verified with mount -a.

## Storage Layout (compute nodes)

### OS Disk (20GB) — Standard Partitions, GPT/UEFI (identical on both)
| Partition | Mount     | Size    | FS                   |
|-----------|-----------|---------|----------------------|
| sda1      | /boot/efi | 600MB   | EFI System Partition |
| sda2      | /boot     | 1GB     | xfs                  |
| sda3      | swap      | 2GB     | swap                 |
| sda4      | /         | ~16.4GB | xfs                  |

No data disk — shared storage via NFS from headnode.

## NFS Configuration

### Exports (headnode)
| Export          | Access | Options             |
|-----------------|--------|---------------------|
| /export/home    | rw     | sync,no_root_squash |
| /export/apps    | ro     | sync                |
| /export/scratch | rw     | sync,no_root_squash |

Exported to: 10.10.10.0/24
Firewall: nfs, mountd, rpc-bind allowed on trusted zone.

### Client Mounts (compute-1 and compute-2)
| Server Export       | Local Mount     | Options          |
|---------------------|-----------------|------------------|
| headnode:/export/home    | /export/home    | defaults,_netdev |
| headnode:/export/apps    | /export/apps    | defaults,_netdev |
| headnode:/export/scratch | /export/scratch | defaults,_netdev |

All in /etc/fstab. Cross-node shared storage verified (file created on
compute-1 visible from compute-2).

## /etc/hosts (all three nodes)
```
127.0.0.1   localhost localhost.localdomain
::1         localhost localhost.localdomain
10.10.10.1   headnode
10.10.10.11  compute-1
10.10.10.12  compute-2
```

## Completed Phases
1. Planning and architecture
2. Hyper-V network setup (LabSwitch1, ClusterSwitch, NAT)
3. Head node VM creation and OS installation
4. Head node post-install (networking, firewall, gateway, LVM, NFS)
5. Compute node deployment (VM creation, OS install, networking, /etc/hosts, NFS mounts)

## Current Phase
About to start:
- Passwordless SSH across the cluster

## Pending Future Phases
- Passwordless SSH
- SLURM scheduler
- User management (FreeIPA/LDAP)
- Environment modules (Lmod)
- Software compilation (OpenMPI, GCC)
- Monitoring (Prometheus + Grafana)
- Automation (Ansible)
- Security hardening

## Key Decisions Made and Why
1. Rocky 9 over Rocky 10 — EPEL maturity, stability
2. Minimal install — lean, intentional, mirrors production
3. Gen 2 (UEFI) over Gen 1 (BIOS) — matches modern hardware, SCSI boot
4. Private switch for cluster — enforces real isolation pattern
5. Internal + NAT for headnode internet — host-managed NAT
6. /28 for NAT subnet — lab needs max 14 hosts
7. /24 for cluster subnet — room for growth
8. Static memory over dynamic — HPC needs predictable resource allocation
9. Standard partitions for OS — no LVM flexibility needed
10. LVM for data disk — need to expand shared storage later
11. xfs over ext4 — better for large files, HPC standard
12. /export convention — signals NFS-exported storage
13. Separate LVs for home/apps/scratch — workload isolation
14. Apps gets most space (20GB) — compilers, MPI, scientific software accumulate
15. NFS exports on cluster subnet only — never exposed to NAT/internet path
16. Apps exported read-only — install once, share everywhere
17. no_root_squash on home/scratch — jobs need root-level file operations
18. Drop-in config files over editing main configs — survives package updates
19. Separate firewall zones per interface — security clarity
20. Dynamic VHDX in lab, Fixed in production — I/O predictability vs space savings
21. CLI for all VM operations — reproducible, documentable, scriptable
22. FAT32 for ESP — universal OS compatibility, diplomatic standard
23. 600MB ESP over 200MB default — headroom for kernel update accumulation
24. _netdev for NFS mounts — prevents boot hang when network isn't ready

## GitHub Repo Structure
```
hpc-lab-hyper-v/
├── README.md
├── docs/
│   ├── 01-planning-and-architecture.md
│   ├── 02-hyperv-network-setup.md
│   ├── 03-headnode-installation.md
│   ├── 04-headnode-post-install.md
│   ├── 05-compute-node-setup.md
│   └── 06-passwordless-ssh.md (next)
├── configs/
│   └── (coming soon)
└── memory/
    └── current_state.md
```

## VM File Locations
```
D:\sanket\VMs\cluster\
├── headnode\
│   ├── headnode-os.vhdx (20GB)
│   └── headnode-data.vhdx (50GB, Dynamic)
├── compute-1\
│   └── compute1-os.vhdx (20GB)
└── compute-2\
    └── compute2-os.vhdx (20GB)
```

## ISO Location
```
D:\sanket\softwares\iso\Rocky-9.7-x86_64-minimal.iso
```
