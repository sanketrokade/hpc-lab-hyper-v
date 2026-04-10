# HPC Lab Cluster (Hyper-V)

Building a fully functional HPC cluster from scratch using Hyper-V VMs
to learn HPC system administration — not just how, but WHY.

## Goal
Learn HPC administration by building, breaking, and understanding
every layer of a cluster from the ground up.

## Lab Environment
- **Host Machine:** HP Z8 Workstation, 65GB RAM, 48 logical CPUs, RTX Ada 6000
- **Hypervisor:** Hyper-V (Windows 11)
- **VM Generation:** Gen 2 (UEFI)
- **OS:** Rocky Linux 9.7 Minimal

## Cluster Architecture

| Node      | Role                     | RAM  | CPUs | Disks                   | Network Adapters           |
|-----------|--------------------------|------|------|-------------------------|----------------------------|
| headnode  | Login + Management + NFS | 8 GB | 4    | 20GB (OS) + 50GB (Data) | LabSwitch1 + ClusterSwitch |
| compute-1 | Compute                  | 4 GB | 4    | 20GB (OS)               | ClusterSwitch only         |
| compute-2 | Compute                  | 4 GB | 4    | 20GB (OS)               | ClusterSwitch only         |

## Network Design

| Switch        | Type     | Subnet          | Purpose                         |
|---------------|----------|-----------------|---------------------------------|
| LabSwitch1    | Internal | 192.168.30.0/28 | Internet access (headnode only) |
| ClusterSwitch | Private  | 10.10.10.0/24   | Internal cluster communication  |

### Static IPs

**LabSwitch1 (NAT):**
- Host gateway: 192.168.30.1
- headnode: 192.168.30.2

**ClusterSwitch (Cluster):**
- headnode: 10.10.10.1
- compute-1: 10.10.10.11
- compute-2: 10.10.10.12

## Key Design Decisions

1. Rocky 9 over Rocky 10 — EPEL maturity, stability
2. Minimal install — lean, intentional, mirrors production
3. Gen 2 (UEFI) over Gen 1 (BIOS) — matches modern hardware
4. Private switch for cluster — enforces real isolation, access only through headnode
5. Internal + NAT for headnode internet — host-managed NAT
6. /28 for NAT subnet — lab doesn't need more than 14 hosts
7. /24 for cluster subnet — room for growth
8. Static memory over dynamic — HPC needs predictable resource allocation
9. Standard partitions for OS disk — no LVM flexibility needed
10. LVM for data disk — need to expand shared storage later
11. xfs over ext4 — better for large files, HPC standard
12. /export convention — signals NFS-exported storage
13. Separate LVs for home/apps/scratch — isolation between workloads
14. Apps gets most space — compilers, MPI, scientific software accumulate
15. NFS exports on cluster subnet only — never exposed to NAT/internet path
16. Apps exported read-only — install once on headnode, shared everywhere
17. no_root_squash on home/scratch — jobs need root-level file operations
18. Drop-in config files over editing main configs — survives package updates
19. Separate firewall zones per interface — security clarity
20. _netdev for NFS mounts — prevents boot hang when network isn't ready
21. Empty passphrase SSH keys — required for non-interactive MPI launches
22. Compute-to-compute SSH trust — MPI spawns processes peer-to-peer
23. Fixed UID 994 for slurm user — must match across all nodes
24. ReturnToService=2 — nodes auto-recover after reboot without manual intervention
25. CoresPerSocket=2 ThreadsPerCore=2 — matches Hyper-V vCPU topology
26. Fixed UID/GID for all users — NFS file ownership requires consistency
27. use_nfs_home_dirs SELinux boolean — allows sshd to read keys on NFS mounts
28. NFS home for regular users — SSH keys work everywhere without copying

## Progress Tracker

- [x] Planning and architecture
- [x] Hyper-V network setup
- [x] Head node VM creation and OS installation
- [x] Head node post-install (networking, firewall, gateway, LVM, NFS)
- [x] Compute node deployment
- [x] NFS client mounts on compute nodes
- [x] Passwordless SSH
- [x] SLURM scheduler
- [x] User management
- [ ] Environment modules
- [ ] Monitoring
- [ ] Automation (Ansible)

## Documentation
Detailed docs for each phase are in the [docs/](docs/) directory.
