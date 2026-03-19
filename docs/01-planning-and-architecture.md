# Phase 1: Planning and Architecture

## Overview
Planning the cluster architecture before touching any VM.
Every choice is intentional and documented with reasoning.

## Hardware Available
- HP Z8 Workstation
- 65 GB RAM
- 48 Logical CPUs (24 cores)
- GPU: NVIDIA RTX Ada 6000
- Windows 11 with Hyper-V

## Resource Allocation

| Node       | RAM  | CPUs | Purpose                |
|------------|------|------|------------------------|
| headnode   | 8 GB | 4    | Login, NFS, Management |
| compute-1  | 4 GB | 4    | Compute jobs           |
| compute-2  | 4 GB | 4    | Compute jobs           |
| **Total**  | 16GB | 12   | Leaves 49GB/36CPUs for host |

## Disk Layout

### headnode
- **Disk 1 (20GB):** OS — GPT/UEFI partitioning
- **Disk 2 (50GB):** Data — LVM for NFS exports

### compute nodes
- **Disk 1 (20GB):** OS only — NFS mounts for shared storage

## Network Architecture

### Why Two Separate Networks?
Real HPC clusters isolate management/external traffic from
high-speed compute traffic. We mirror this with two virtual switches.

### Switch Design
| Switch        | Type     | Subnet         | Why This Type?                          |
|---------------|----------|----------------|-----------------------------------------|
| LabSwitch1    | Internal | 192.168.30.0/28| Headnode needs internet via host NAT    |
| ClusterSwitch | Private  | 10.10.10.0/24  | Compute nodes isolated, no host access  |

### Why Private over Internal for cluster?
- Internal lets the Windows host reach compute nodes directly
- Private enforces the real-world pattern: all access through headnode
- Mirrors production security: compute nodes are never directly reachable

### IP Assignments
- headnode LabSwitch1: 192.168.30.2 (gateway: 192.168.30.1)
- headnode ClusterSwitch: 10.10.10.1
- compute-1 ClusterSwitch: 10.10.10.11
- compute-2 ClusterSwitch: 10.10.10.12

## OS Choice: Rocky Linux 9 Minimal
- RHEL-based: matches enterprise HPC environments
- Minimal: install only what we need, understand every package
- Rocky 9 over 10: EPEL and ecosystem maturity
