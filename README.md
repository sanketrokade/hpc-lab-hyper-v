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

| Node                       | Role                     | RAM  | CPUs | Disks                   | Network Adapters           |
|----------------------------|--------------------------|------|------|-------------------------|----------------------------|
| headnode.hpclab.internal   | Login + Mgmt + NFS + IPA | 8 GB | 4    | 20GB (OS) + 50GB (Data) | LabSwitch1 + ClusterSwitch |
| compute-1.hpclab.internal  | Compute                  | 4 GB | 4    | 20GB (OS)               | ClusterSwitch only         |
| compute-2.hpclab.internal  | Compute                  | 4 GB | 4    | 20GB (OS)               | ClusterSwitch only         |

All nodes use FQDN hostnames as required by FreeIPA.

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

## Identity & Authentication

After Phase 14, the cluster runs as a single FreeIPA realm:

- **Realm:** `HPCLAB.INTERNAL`
- **Domain:** `hpclab.internal`
- **IPA Server:** headnode.hpclab.internal (single-master)
- **DNS:** IPA-managed BIND, forwards external queries to 8.8.8.8 / 1.1.1.1
- **Authentication:** Kerberos (with auto-acquired TGTs at login)
- **Identity:** LDAP via SSSD on every node
- **Compute-to-compute SSH:** Kerberos GSSAPI — no SSH keys for users

## Key Design Decisions

### Hypervisor & OS
1. Rocky 9 over Rocky 10 — EPEL maturity, stability
2. Minimal install — lean, intentional, mirrors production
3. Gen 2 (UEFI) over Gen 1 (BIOS) — matches modern hardware
4. CLI for all VM operations — reproducible, documentable, scriptable

### Networking
5. Private switch for cluster — enforces real isolation, access only through headnode
6. Internal + NAT for headnode internet — host-managed NAT
7. /28 for NAT subnet — lab doesn't need more than 14 hosts
8. /24 for cluster subnet — room for growth
9. Separate firewall zones per interface — security clarity

### Storage
10. Standard partitions for OS disk — no LVM flexibility needed
11. LVM for data disk — need to expand shared storage later
12. xfs over ext4 — better for large files, HPC standard
13. /export convention — signals NFS-exported storage
14. Separate LVs for home/apps/scratch — isolation between workloads
15. Apps gets most space — compilers, MPI, scientific software accumulate
16. Mode 1777 on /export/scratch — sticky-bit world-writable like /tmp
17. NFS exports on cluster subnet only — never exposed to NAT/internet path
18. Apps exported read-only — install once on headnode, shared everywhere
19. no_root_squash on home/scratch — jobs need root-level file operations

### Configuration Discipline
20. Drop-in config files over editing main configs — survives package updates
21. Backup vendor configs before editing — useful reference, never delete
22. _netdev for NFS mounts — prevents boot hang when network isn't ready

### SLURM
23. Fixed UID 994 for slurm user — must match across all nodes
24. ReturnToService=2 — nodes auto-recover after reboot without manual intervention
25. CoresPerSocket=2 ThreadsPerCore=2 — matches Hyper-V vCPU topology
26. slurmdbd before slurmctld startup order — controller connects to DB on start

### Modules & Software
27. Lmod over Environment Modules — Lua-based, faster, dependency resolution
28. Modulefiles in /export/apps/modulefiles — alongside software, NFS shared

### Monitoring
29. Prometheus LTS 3.5.2 — stability over latest features
30. Grafana on public zone — reachable from Windows host browser
31. Node Exporter Full dashboard — most widely used, 20M+ downloads

### Automation
32. Ansible on headnode only — agentless, uses existing SSH
33. Idempotent modules — state-checking, safe to rerun anytime

### MPI
34. OpenMPI 4.1.8 from source — version control, install to /export/apps
35. No PMIx compile flag — SLURM 22.05 built without PMI support
36. mpirun --mca plm rsh — SSH-based launch bypasses SLURM PMI limitation
37. --prefix flag on mpirun — compute nodes find orted without PATH changes
38. eth0 trusted zone on compute nodes — private network, no port restrictions
39. Jobs submitted as regular user — OpenMPI refuses to run as root

### Time Synchronization
40. Disable Hyper-V Time Synchronization on guests — prevents host clock from fighting chronyd
41. chrony over ntpd — designed for VMs, default on RHEL 8+
42. Hierarchical NTP (headnode → compute) — single source of truth, matches air-gapped HPC pattern
43. NTP served inward only via `allow 10.10.10.0/24` — never outward to NAT
44. `makestep 1.0 3` — step only at startup, slew thereafter to protect running services
45. No external NTP fallback on compute nodes — consistent cluster time over independently-correct time

### FreeIPA / Identity
46. Realm `HPCLAB.INTERNAL` (`.internal` TLD) — RFC 8375 reserved for private use
47. FQDN hostnames cluster-wide — IPA requirement, production standard
48. Short-name aliases retained in /etc/hosts — backwards compatibility
49. Pre-seed FQDN known_hosts before IPA install — chained SSH (MPI) needs FQDN host keys
50. Integrated DNS in IPA — automatic SRV records, production HPC pattern
51. `--ip-address=10.10.10.1` at install — IPA serves identity to cluster only, not NAT
52. `--no-reverse` then add reverse zone deliberately — controlled DNS architecture
53. Add A/PTR records to IPA before enrolling clients — DNS as source of truth
54. `--no-ntp` on client install — preserve Phase 13 chrony setup
55. Recreate users in IPA, not migrate — clean break, avoid UID collisions
56. `--random` passwords for new users — strong by default, change on first login
57. NFS home (`/export/home/<user>`) for HPC users — matches existing storage layout
58. `admin` keeps local home (`/home/admin`) — emergency-fix scenarios independent of NFS
59. Kerberos GSSAPI replaces user SSH keys — no key distribution for HPC users

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
- [x] Environment modules
- [x] Monitoring
- [x] Automation (Ansible)
- [x] OpenMPI 4.1.8 — multi-node MPI jobs verified
- [x] Clock synchronization (chrony, hierarchical NTP)
- [x] FreeIPA / Centralized user management — Kerberos SSO + LDAP + IPA DNS

## Upcoming
- Security hardening (HBAC, sudo policies, ssh-trust-dns, sshd hardening)

## Known Issues
- `mkhomedir` does not fire on headnode logins (works on compute nodes;
  homes get created on first compute-node login). Defer to security
  hardening phase.

## Documentation
Detailed docs for each phase are in the [docs/](docs/) directory.
