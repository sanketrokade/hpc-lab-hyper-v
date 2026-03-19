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
- **OS:** Rocky Linux 9 Minimal

## Cluster Architecture

| Node       | Role                     | RAM  | CPUs | Disks                 | Network Adapters              |
|------------|--------------------------|------|------|-----------------------|-------------------------------|
| headnode   | Login + Management + NFS | 8 GB | 4    | 20GB (OS) + 50GB (Data) | LabSwitch1 + ClusterSwitch  |
| compute-1  | Compute                  | 4 GB | 4    | 20GB (OS)             | ClusterSwitch only            |
| compute-2  | Compute                  | 4 GB | 4    | 20GB (OS)             | ClusterSwitch only            |

## Network Design

| Switch        | Type     | Subnet            | Purpose                          |
|---------------|----------|--------------------|----------------------------------|
| LabSwitch1    | Internal | 192.168.30.0/28    | Internet access (headnode only)  |
| ClusterSwitch | Private  | 10.10.10.0/24      | Internal cluster communication   |

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
4. Private switch for cluster — enforces real isolation pattern
5. Internal + NAT for headnode internet — host-managed NAT
6. /28 for NAT subnet — lab doesn't need more than 14 hosts
7. /24 for cluster subnet — room for growth

## Progress Tracker

- [x] Planning and architecture
- [x] Hyper-V network setup
- [ ] Head node VM creation and OS installation
- [ ] Head node post-install configuration
- [ ] Gateway and NFS setup
- [ ] Compute node deployment
- [ ] SLURM scheduler
- [ ] User management
- [ ] Environment modules
- [ ] Monitoring
- [ ] Automation (Ansible)

## Documentation
Detailed docs for each phase are in the [docs/](docs/) directory.
