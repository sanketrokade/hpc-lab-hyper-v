# HPC Lab Cluster — Conversation State
# Last Updated: Phase 14 Complete (FreeIPA / Centralized User Management)

## Who Am I Working With
- Name: Sanket
- Goal: Become an HPC System Administrator — not average, the best
- Mindset: Practical, evidence-based, wants to understand WHY not just HOW
- Skill Level: Comfortable with Linux (RHEL and Debian), used VMware and Hyper-V
- Previous HPC lab experience on VMware (completed through Phase 5)
- Rebuilding from scratch on Hyper-V for deeper learning
- Ansible installed and tested but not yet used in workflows — wants to
  build manually first, then identify automation opportunities

## Rules
1. Keep it conversational — no info dumps
2. After each milestone, provide raw documentation files for GitHub (one file per message)
3. Maintain this state file — update after each phase
4. Explain the WHY and history behind every technology choice
5. Challenge Sanket's thinking, don't just give answers
6. All VM operations via PowerShell CLI — not GUI
7. Documentation must be properly formatted markdown, copy-paste ready
8. Do not take decisions without asking — provide suggestions but keep them brief
9. Do not start a new conversation mid-phase
10. Manual implementation first; flag Ansible opportunities for discussion

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

| Node                       | RAM | CPUs | Disks               | Network                    |
|----------------------------|-----|------|---------------------|----------------------------|
| headnode.hpclab.internal   | 8GB | 4    | 20GB OS + 50GB Data | LabSwitch1 + ClusterSwitch |
| compute-1.hpclab.internal  | 4GB | 4    | 20GB OS             | ClusterSwitch only         |
| compute-2.hpclab.internal  | 4GB | 4    | 20GB OS             | ClusterSwitch only         |

Total used: 16GB RAM, 12 CPUs — leaves 49GB and 36 CPUs for host.

All nodes use FQDN as system hostname (set via `hostnamectl`). Short names
remain as aliases in `/etc/hosts` for backwards compatibility.

## Hyper-V Virtual Switches

| Switch         | Type     | Purpose                         |
|----------------|----------|---------------------------------|
| vSwitch1       | External | Host internet (not used by VMs) |
| LabSwitch1     | Internal | Headnode internet via NAT       |
| ClusterSwitch  | Private  | Cluster internal only           |
| Default Switch | Internal | Hyper-V default (ignored)       |

## Hyper-V Integration Services

Time Synchronization is **disabled** on all three VMs to prevent the
Hyper-V host clock from fighting chronyd. All other integration services
remain at default.

## Host-Side NAT Configuration
- LabSwitch1 host IP: 192.168.30.1/28
- NetNat name: LabNAT
- NAT prefix: 192.168.30.0/28

## Network Design

| Network       | Subnet          | Purpose              |
|---------------|-----------------|----------------------|
| LabSwitch1    | 192.168.30.0/28 | Internet (head only) |
| ClusterSwitch | 10.10.10.0/24   | Cluster internal     |

### Static IPs
- headnode LabSwitch1: 192.168.30.2/28 (gateway: 192.168.30.1)
- headnode ClusterSwitch: 10.10.10.1/24
- compute-1 ClusterSwitch: 10.10.10.11/24 (gateway: 10.10.10.1)
- compute-2 ClusterSwitch: 10.10.10.12/24 (gateway: 10.10.10.1)

### Interfaces on headnode
- eth0: LabSwitch1, Static (192.168.30.2), MAC 00:15:5d:59:08:03, firewall public zone
- eth1: ClusterSwitch, Static (10.10.10.1), MAC 00:15:5d:59:08:04, firewall trusted zone

### Interfaces on compute-1
- eth0: ClusterSwitch, Static (10.10.10.11), MAC 00:15:5d:59:08:08, firewall trusted zone
- Connection profile name: cluster

### Interfaces on compute-2
- eth0: ClusterSwitch, Static (10.10.10.12), MAC 00:15:5d:59:08:09, firewall trusted zone
- Connection profile name: cluster

### Gateway Configuration
- ip_forward = 1 (set in /etc/sysctl.d/99-cluster.conf on headnode)
- Masquerading enabled on public zone
- Compute nodes use 10.10.10.1 as default gateway
- DNS: compute nodes point at 10.10.10.1 (IPA's BIND), with 8.8.8.8 fallback

## DNS Configuration

### Resolver Order (compute nodes)
1. `10.10.10.1` — IPA's BIND (knows internal names, forwards external)
2. `8.8.8.8` — pure fallback for external names if headnode is down
- Search domain: `hpclab.internal`

### Authoritative Zones (IPA-managed BIND)
| Zone                       | Type         | Purpose                          |
|----------------------------|--------------|----------------------------------|
| hpclab.internal            | Forward      | All internal cluster names       |
| 10.10.10.in-addr.arpa.     | Reverse      | PTR records for cluster subnet   |

### Forwarders (for external queries)
- 8.8.8.8 (Google)
- 1.1.1.1 (Cloudflare)

### DNS Records
| Name                          | Type | Value                          |
|-------------------------------|------|--------------------------------|
| headnode.hpclab.internal      | A    | 10.10.10.1                     |
| compute-1.hpclab.internal     | A    | 10.10.10.11                    |
| compute-2.hpclab.internal     | A    | 10.10.10.12                    |
| ipa-ca.hpclab.internal        | A    | 10.10.10.1                     |
| 1.10.10.10.in-addr.arpa.      | PTR  | headnode.hpclab.internal.      |
| 11.10.10.10.in-addr.arpa.     | PTR  | compute-1.hpclab.internal.     |
| 12.10.10.10.in-addr.arpa.     | PTR  | compute-2.hpclab.internal.     |
| _kerberos._tcp.hpclab.internal| SRV  | 0 100 88 headnode...           |
| _ldap._tcp.hpclab.internal    | SRV  | 0 100 389 headnode...          |
| _kpasswd._tcp.hpclab.internal | SRV  | 0 100 464 headnode...          |
| compute-1, compute-2, headnode| SSHFP| Auto-published host SSH keys   |

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

| LV         | Size | Mount           | FS  | Mode   |
|------------|------|-----------------|-----|--------|
| lv_home    | 10GB | /export/home    | xfs | 755    |
| lv_apps    | 20GB | /export/apps    | xfs | 755    |
| lv_scratch | 15GB | /export/scratch | xfs | 1777   |

`/export/scratch` mode is **1777** (sticky bit + world-writable, like
`/tmp`) so any user can write while file deletion is restricted to owner.
This was corrected in Phase 14 from the original 755.

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
| Server Export            | Local Mount     | Options          |
|--------------------------|-----------------|------------------|
| headnode:/export/home    | /export/home    | defaults,_netdev |
| headnode:/export/apps    | /export/apps    | defaults,_netdev |
| headnode:/export/scratch | /export/scratch | defaults,_netdev |

All in /etc/fstab. Cross-node shared storage verified.

## Time Synchronization (chrony)

### Architecture
```
Internet NTP (Cloudflare/pool.ntp.org)
            ↓
    headnode (stratum 4)
            ↓
    compute-1, compute-2 (stratum 5)
```

### Services Per Node
| Node      | Role            | Source            | Stratum |
|-----------|-----------------|-------------------|---------|
| headnode  | Server + Client | pool.ntp.org      | 4       |
| compute-1 | Client only     | headnode          | 5       |
| compute-2 | Client only     | headnode          | 5       |

### Verified State
- Headnode: ~3 ms offset from internet NTP
- Compute-1: ~50 µs offset from headnode
- Compute-2: ~6 µs offset from headnode
- Kerberos 5-minute tolerance has 5,000,000× headroom

## FreeIPA Configuration

### Realm and Domain
- Kerberos realm: `HPCLAB.INTERNAL`
- DNS domain: `hpclab.internal`
- IPA version: 4.12.2-22.el9_7.4
- Server: `headnode.hpclab.internal`
- Single-master (no replicas)

### IPA Server Services (headnode)
| Service          | Port               | Purpose              |
|------------------|--------------------|----------------------|
| Directory Service| 389/tcp, 636/tcp   | LDAP                 |
| krb5kdc          | 88/tcp, 88/udp     | Kerberos KDC         |
| kadmin           | 464/tcp, 464/udp, 749/tcp | Kerberos admin |
| named (BIND)     | 53/tcp, 53/udp     | Realm DNS + forwarding |
| httpd            | 80/tcp, 443/tcp    | Web UI               |
| pki-tomcatd      | 8443/tcp (internal)| Certificate Authority|
| ipa-custodia     | —                  | Secret/key management|
| ipa-otpd         | —                  | One-time password    |
| ipa-dnskeysyncd  | —                  | DNSSEC key sync      |

### IPA Client Services (all nodes)
| Service    | Purpose                          |
|------------|----------------------------------|
| sssd       | Identity/auth via LDAP+Kerberos  |
| oddjobd    | mkhomedir helper                 |
| certmonger | Certificate renewal              |

### Firewall Posture
- All IPA ports accessible on **eth1 (cluster, trusted zone)** via
  `target: ACCEPT`
- **NO** IPA ports exposed on eth0 (NAT, public zone)
- Daemons bind wildcard `0.0.0.0`; firewalld enforces the boundary

### Web UI
- URL: `https://headnode.hpclab.internal/ipa/ui/`
- Reachable from cluster network only
- Self-signed CA cert (IPA's internal CA)

### Hosts in Realm
| Hostname                    | Principal                                     |
|-----------------------------|-----------------------------------------------|
| headnode.hpclab.internal    | host/headnode.hpclab.internal@HPCLAB.INTERNAL |
| compute-1.hpclab.internal   | host/compute-1.hpclab.internal@HPCLAB.INTERNAL|
| compute-2.hpclab.internal   | host/compute-2.hpclab.internal@HPCLAB.INTERNAL|

### Users in IPA
| Username | UID       | GID       | Home                  | Shell     | Type     |
|----------|-----------|-----------|-----------------------|-----------|----------|
| admin    | 536000000 | 536000000 | /home/admin (local)   | /bin/bash | IPA admin|
| hpcuser1 | 536000004 | 536000004 | /export/home/hpcuser1 | /bin/bash | HPC user |
| hpcuser2 | 536000005 | 536000005 | /export/home/hpcuser2 | /bin/bash | HPC user |

### Groups in IPA
| Group     | GID       | Members             | Type        |
|-----------|-----------|---------------------|-------------|
| hpcusers  | 536000003 | hpcuser1, hpcuser2  | Custom      |
| ipausers  | (none)    | hpcuser1, hpcuser2  | Default     |
| admins    | 536000001 | admin               | IPA admins  |

### Password Policy
- Default IPA policy: minimum 8 chars, 3 character classes
- `--random` users have one-time password, must change on first login
- Password expiration enforced

## Authentication Flow (After IPA)

### User SSH Login
1. User runs `ssh hpcuser1@compute-1.hpclab.internal`
2. sshd uses PAM via SSSD
3. SSSD queries IPA LDAP over Kerberos for user info
4. PAM `pam_krb5` validates password against KDC
5. PAM `pam_oddjob_mkhomedir` creates home if missing (compute nodes only)
6. PAM acquires Kerberos TGT for user
7. User dropped at shell with `KCM:UID:N` ticket cache populated

### Compute-to-Compute SSH (Kerberos GSSAPI)
1. hpcuser1 on compute-1 runs `ssh compute-2 hostname`
2. SSH client uses existing TGT (no password prompt)
3. KDC issues service ticket `host/compute-2.hpclab.internal@HPCLAB.INTERNAL`
4. compute-2's sshd validates ticket against its host keytab
5. User authenticated; no SSH keys involved

### NFS Identity
- All nodes resolve hpcuser1 → UID 536000004 via SSSD/LDAP
- File ownership stays consistent across cluster automatically
- No more manual UID pinning required

## Passwordless SSH (Root)

### Trust Matrix (root)
| From \ To | headnode | compute-1 | compute-2 |
|-----------|----------|-----------|-----------|
| headnode  | ✓        | ✓         | ✓         |
| compute-1 | ✓        | ✓ (self)  | ✓         |
| compute-2 | ✓        | ✓         | ✓ (self)  |

All root keys are 4096-bit RSA, no passphrase.
Keys stored in /root/.ssh/ (local disk per node).
Self-SSH for compute nodes added in Phase 14 for MPI launch patterns.

### Regular Users
- No SSH keys required — Kerberos GSSAPI handles authentication
- TGT acquired automatically at login by PAM
- Service tickets for individual hosts acquired on demand
- `~/.ssh/known_hosts` populated automatically via IPA SSHFP records

## SLURM Configuration

### Services Per Node
| Service   | Node      | Port | Status  |
|-----------|-----------|------|---------|
| munge     | all nodes | —    | running |
| mariadb   | headnode  | 3306 | running |
| slurmdbd  | headnode  | 6819 | running |
| slurmctld | headnode  | 6817 | running |
| slurmd    | compute-* | 6818 | running |

### Port Reference
| Port | Daemon    | Open on   |
|------|-----------|-----------|
| 6817 | slurmctld | headnode  |
| 6818 | slurmd    | compute-* |
| 6819 | slurmdbd  | headnode  |

### slurm.conf Key Settings
- ClusterName: hpc-lab
- SlurmctldHost: headnode (resolves via /etc/hosts to 10.10.10.1)
- SchedulerType: sched/backfill
- SelectType: select/cons_tres
- SelectTypeParameters: CR_Core_Memory
- ReturnToService: 2
- AccountingStorageType: accounting_storage/slurmdbd
- JobAcctGatherType: jobacct_gather/cgroup

### Node Definitions
```
NodeName=compute-1 NodeAddr=10.10.10.11 CPUs=4 RealMemory=3500
  Sockets=1 CoresPerSocket=2 ThreadsPerCore=2 State=UNKNOWN
NodeName=compute-2 NodeAddr=10.10.10.12 CPUs=4 RealMemory=3500
  Sockets=1 CoresPerSocket=2 ThreadsPerCore=2 State=UNKNOWN
```

### Partition
```
PartitionName=compute Nodes=compute-1,compute-2 Default=YES
  MaxTime=INFINITE State=UP
```

### Verified With IPA Users
- SLURM accepts IPA users (UID 536000004, 536000005) without modification
- Job output files written to NFS home, owned by IPA UID
- `id` inside job shows full IPA UID/GID, including `hpcusers` membership

## SELinux Requirements
- use_nfs_home_dirs=1 on all compute nodes
  (still required for SSH key auth on legacy paths)

## Authselect Profiles

| Node      | Profile | Features                      |
|-----------|---------|-------------------------------|
| headnode  | sssd    | with-sudo, with-mkhomedir     |
| compute-1 | sssd    | with-sudo, with-mkhomedir     |
| compute-2 | sssd    | with-sudo, with-mkhomedir     |

NOTE: Despite identical authselect config, mkhomedir on headnode does not
fire automatically — known issue, see "Known Issues" below.

## Environment Modules (Lmod)

### Installation
- Lmod 8.7.65 installed on all three nodes via EPEL
- Initialized via /etc/profile.d/modules.sh (automatic on login)
- Custom modulepath: /etc/profile.d/01-cluster-modulepath.sh

### Modulefiles Created
| Module        | Location                                   | Software Path             |
|---------------|--------------------------------------------|---------------------------|
| gcc/11.5.0    | /export/apps/modulefiles/gcc/11.5.0.lua    | /usr (system)             |
| openmpi/4.1.8 | /export/apps/modulefiles/openmpi/4.1.8.lua | /export/apps/openmpi/4.1.8|

## Monitoring Stack

### Services
| Service        | Node     | Port | Status  | Access           |
|----------------|----------|------|---------|------------------|
| prometheus     | headnode | 9090 | running | trusted + public |
| node_exporter  | all      | 9100 | running | trusted zone     |
| grafana-server | headnode | 3000 | running | public zone      |

### Access
- Grafana: http://192.168.30.2:3000
- Prometheus: http://192.168.30.2:9090
- Dashboard: Node Exporter Full (ID: 1860)

## Ansible Automation

### Installation
- Ansible core 2.14.18 installed on headnode only
- Location: /usr/bin/ansible
- Project directory: /etc/ansible/hpc-lab

### Project Structure
```
/etc/ansible/hpc-lab/
├── ansible.cfg
├── inventory/
│   └── hosts
├── group_vars/
├── host_vars/
├── roles/
└── playbooks/
    ├── cluster-facts.yml
    ├── add-user.yml
    └── cluster-health.yml
```

### Inventory Groups
| Group    | Members               | Purpose                    |
|----------|-----------------------|----------------------------|
| headnodes| headnode              | Headnode-specific tasks    |
| compute  | compute-1, compute-2  | Compute-specific tasks     |
| cluster  | all three nodes       | Cluster-wide tasks         |

NOTE: Inventory still uses short hostnames; works because /etc/hosts has
short-name aliases. Should be updated to FQDNs in a future refresh.

## OpenMPI

### Installation
- Version: 4.1.8
- Source: https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.8.tar.gz
- Install path: /export/apps/openmpi/4.1.8 (NFS shared)
- Configure flags: --prefix only (no PMIx — SLURM built without PMI support)
- Modulefile: /export/apps/modulefiles/openmpi/4.1.8.lua

### Launch Method
- SLURM 22.05 from Rocky repos built without PMI/PMIx
- srun cannot bootstrap MPI processes
- Use: mpirun --mca plm rsh --prefix /export/apps/openmpi/4.1.8
- --prefix ensures compute nodes find orted without manual PATH setup
- SSH between compute nodes uses Kerberos GSSAPI (no SSH keys for users)

### Firewall
- compute-1 and compute-2 eth0 in trusted zone
- Required for arbitrary TCP ports between MPI daemons

### Verified Working Under IPA
- 4 ranks across 2 nodes (2 per node)
- Jobs submitted as hpcuser1 (IPA user, UID 536000004)
- No SSH keys, no munge tricks, no host file edits — Kerberos handles auth
- Output shows IPA UID and FQDN hostnames

### Job Script Template
```bash
#!/bin/bash
#SBATCH --job-name=mpi-test
#SBATCH --nodes=2
#SBATCH --ntasks=4
#SBATCH --partition=compute

module load openmpi/4.1.8
mpirun --mca plm rsh \
       --prefix /export/apps/openmpi/4.1.8 \
       -np 4 \
       --host compute-1:2,compute-2:2 \
       /path/to/binary
```

## /etc/hosts (all three nodes)
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.10.10.1   headnode.hpclab.internal headnode
10.10.10.11  compute-1.hpclab.internal compute-1
10.10.10.12  compute-2.hpclab.internal compute-2
```

FQDN is canonical; short names are aliases for backwards compatibility.
Pre-IPA versions backed up to `/etc/hosts.pre-ipa` on each node.

## Completed Phases
1. Planning and architecture
2. Hyper-V network setup (LabSwitch1, ClusterSwitch, NAT)
3. Head node VM creation and OS installation
4. Head node post-install (networking, firewall, gateway, LVM, NFS)
5. Compute node deployment (VM creation, OS install, networking, NFS mounts)
6. Passwordless SSH (root access across all nodes)
7. SLURM scheduler (munge, MariaDB, slurmctld, slurmdbd, slurmd)
8. User management (local users with NFS home, passwordless SSH, SLURM jobs)
9. Environment modules (Lmod, gcc/11.5.0 modulefile, SLURM job with module)
10. Monitoring (Prometheus, Node Exporter, Grafana, Node Exporter Full dashboard)
11. Automation (Ansible inventory, facts, user creation, health check playbooks)
12. OpenMPI (4.1.8 from source, SSH-based launch, multi-node MPI jobs verified)
13. Clock synchronization (chrony, hierarchical NTP, FreeIPA prerequisite)
14. FreeIPA / Centralized user management (realm HPCLAB.INTERNAL,
    BIND DNS, Kerberos SSO, IPA users, end-to-end SLURM+MPI verified)

## Pending Future Phases
- Phase 15: Security hardening (HBAC, sudo policies, ssh-trust-dns,
  resolve headnode mkhomedir issue, firewall audit, sshd hardening)

## Known Issues (Carried Forward)

| Issue | Impact | Notes |
|-------|--------|-------|
| `mkhomedir` does not fire on headnode logins | First login on headnode lands at `/`; subsequent logins work after compute node creates home on NFS | All preconditions appear correct (PAM stack, oddjobd, NFS perms, SELinux). Defer to Phase 15. |
| No reverse zone for `192.168.30.0/28` | NAT-side reverse lookups fail | Not currently needed; add if future tooling requires it |
| `ipausers` group has no GID | Not visible to `getent group ipausers` | IPA design choice for default access group; harmless |
| Ansible inventory uses short hostnames | Works via /etc/hosts aliases | Refresh inventory to FQDNs in future cleanup |

## Active Snapshots
| Snapshot          | Date               | Purpose                              |
|-------------------|--------------------|--------------------------------------|
| pre-freeipa       | April 16, 2026     | Deep history (failed FreeIPA attempt)|
| mpi-working       | April 27, 2026 AM  | Post-MPI clean state                 |
| post-clock-sync   | April 27, 2026 PM  | Pre-FreeIPA, time-synced             |
| post-freeipa      | April 27, 2026 PM  | FreeIPA functional, end-to-end verified |

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
25. Empty passphrase on SSH keys — required for non-interactive MPI launches
26. Compute-to-compute SSH trust — MPI spawns processes peer-to-peer
27. Fixed UID 994 for slurm user — must match across all nodes
28. ReturnToService=2 — nodes auto-recover after reboot without manual intervention
29. CoresPerSocket=2 ThreadsPerCore=2 — matches Hyper-V vCPU topology
30. slurmdbd before slurmctld startup order — controller connects to DB on start
31. Fixed UID/GID for all users (pre-IPA) — NFS file ownership requires consistency
32. -M flag on compute nodes — prevent duplicate home directory creation (pre-IPA)
33. use_nfs_home_dirs SELinux boolean — allows sshd to read keys on NFS mounts
34. NFS home for regular users — SSH keys / Kerberos work everywhere
35. Lmod over Environment Modules — Lua-based, faster, dependency resolution
36. Modulefiles in /export/apps/modulefiles — alongside software, NFS shared
37. /etc/profile.d/ for Lmod init — automatic for all users on login
38. Prometheus LTS 3.5.2 — stability over latest features
39. 30 day retention — sufficient for lab, avoids disk bloat
40. Grafana on public zone — must be reachable from Windows host browser
41. Prometheus on trusted zone — internal cluster use only
42. Node Exporter Full dashboard ID 1860 — most widely used, 20M+ downloads
43. cluster label in prometheus.yml — tags all metrics with cluster name
44. Ansible on headnode only — agentless, uses existing SSH
45. host_key_checking=False — no prompts on trusted cluster network
46. ControlPersist=60s — reuse SSH connections, faster playbook execution
47. headnodes group (plural) — avoids host/group name conflict warning
48. Idempotent modules over shell/command — state-checking, safe to rerun
49. OpenMPI 4.1.8 from source — version control, install to /export/apps
50. No PMIx compile flag — SLURM 22.05 built without PMI support
51. mpirun --mca plm rsh — SSH-based launch bypasses broken SLURM PMI
52. --prefix flag on mpirun — compute nodes find orted without PATH changes
53. eth0 trusted zone on compute nodes — private network, no port restrictions
54. Jobs as hpcuser1 — OpenMPI refuses to run MPI as root
55. Disable Hyper-V Time Synchronization on guests — prevents host clock from fighting chronyd
56. chrony over ntpd — designed for VMs, default on RHEL 8+, handles clock jumps
57. Hierarchical NTP (headnode → compute) — single source of truth, matches air-gapped HPC pattern
58. allow 10.10.10.0/24 only on headnode chrony — NTP served inward only
59. NTP firewall on trusted zone only — no NAT-side exposure
60. makestep 1.0 3 — step at startup only, slew thereafter to protect running services
61. rtcsync enabled — hardware clock stays correct across reboots
62. server (not pool) on compute nodes — exactly one explicit source
63. No external NTP fallback on compute nodes — consistent cluster time over independently-correct time
64. Backup vendor configs before editing — chrony.conf.original retained for reference
65. Realm `HPCLAB.INTERNAL` (`.internal` TLD) — RFC 8375 reserved, no public collision
66. FQDN hostnames cluster-wide — IPA hard requirement, production standard
67. Short-name aliases retained in /etc/hosts — backwards compatibility for existing scripts
68. Pre-seed FQDN known_hosts before IPA install — chained SSH (MPI) needs FQDN host keys
69. Self-SSH key added on compute nodes — some MPI launch patterns require it
70. mode 1777 on /export/scratch — sticky-bit world-writable like /tmp; latent bug from Phase 4 fixed
71. `--no-reverse` at IPA server install, add reverse zone deliberately — controlled DNS architecture
72. `--ip-address=10.10.10.1` (cluster IP, not NAT) — IPA serves identity to cluster only
73. Integrated DNS in IPA — production HPC pattern, automatic SRV records for service discovery
74. `--no-ntp` on client install — preserve Phase 13 chrony setup intentionally
75. Add A/PTR records to IPA DNS before enrolling clients — DNS as source of truth, no enrollment warnings
76. Recreate users in IPA, not migrate — clean break, avoid UID-range collisions with IPA's 536000000+
77. `--random` passwords for new IPA users — strong by default, force change on first login
78. NFS home (`/export/home/<user>`) for HPC users via `--homedir` — matches existing storage layout
79. `admin` keeps local home (`/home/admin`) — emergency-fix scenarios independent of NFS
80. `sss_cache -E` after user changes — force fresh LDAP queries, eliminate stale negative-cache
81. Kerberos GSSAPI replaces user SSH keys — IPA-installed `04-ipa.conf` enables this cluster-wide

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
│   ├── 06-passwordless-ssh.md
│   ├── 07-slurm-setup.md
│   ├── 08-user-management.md
│   ├── 09-environment-modules.md
│   ├── 10-monitoring.md
│   ├── 11-ansible-automation.md
│   ├── 12-openmpi.md
│   ├── 13-clock-sync.md
│   └── 14-freeipa.md
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
