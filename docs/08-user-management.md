# Phase 8: User Management

## Overview
Create cluster users that work seamlessly across all nodes — same UID/GID
everywhere, home directories on NFS shared storage, passwordless SSH via
shared keys, and ability to submit SLURM jobs.

## Design Decisions

### Why Local Users (Not LDAP/FreeIPA) Now
Centralized user management (FreeIPA, LDAP, Active Directory) is the
production standard for clusters with many users. We use local users here
because:
- Simpler to understand and debug at this stage
- SLURM must be stable before adding authentication infrastructure
- FreeIPA brings DNS, Kerberos, LDAP, CA — too many moving parts at once
- FreeIPA/LDAP is a planned future phase

### User Design Principles
- **Fixed UIDs** — must be identical across all nodes
- **NFS home directories** — `/export/home/username`, shared automatically
- **Group-based access control** — assign permissions to groups, not individuals
- **No sudo** — regular users have no administrative privileges
- **Service accounts separate** — never mix login users with daemon accounts

### Why Fixed UIDs Matter
When compute-1 looks at a file owned by UID 1001 on the NFS mount, it checks
its local `/etc/passwd` to resolve the name. If UID 1001 maps to a different
user on compute-1, that user owns your files. If UID 1001 doesn't exist,
files show as owned by a number — no name. Jobs writing output files, reading
input data — all break in subtle ways.

### NFS Home Directories
Home on `/export/home` means:
- User logs into any node — same files, same environment
- SSH keys generated once — work everywhere (same `authorized_keys`)
- Job output written on compute node — immediately visible on headnode
- No syncing, no copying — one filesystem, all nodes

## Prerequisites
- NFS exports configured (Phase 5 complete)
- SLURM working (Phase 7 complete)
- All nodes reachable from headnode

## 8.1 Create Group and User

### Create Group First (all nodes, same GID)

**On headnode:**
```bash
groupadd -g 1001 hpcusers
```

**On compute-1 and compute-2:**
```bash
groupadd -g 1001 hpcusers
```

### Create User on Headnode (with home directory)

```bash
useradd -u 1001 -g 1001 \
  -d /export/home/hpcuser1 \
  -m \
  -s /bin/bash \
  -c "HPC Test User" \
  hpcuser1

passwd hpcuser1
```

**Flag explanation:**
- `-u 1001` — explicit UID, must match all nodes
- `-g 1001` — primary group (hpcusers)
- `-d /export/home/hpcuser1` — home on NFS, not /home
- `-m` — create home directory
- `-s /bin/bash` — login shell
- `-M` flag on compute nodes — do NOT create home (it already exists on NFS)

### Create User on Compute Nodes (no home directory creation)

**On compute-1 and compute-2:**
```bash
useradd -u 1001 -g 1001 \
  -d /export/home/hpcuser1 \
  -s /bin/bash \
  -c "HPC Test User" \
  -M \
  hpcuser1
```

`-M` explicitly prevents home directory creation. Without it, useradd
might try to create `/export/home/hpcuser1` locally and conflict with
the NFS mount.

### Verify Identical Records on All Nodes

```bash
getent passwd hpcuser1
getent group hpcusers
```

Must be identical on headnode, compute-1, and compute-2:
```
hpcuser1:x:1001:1001:HPC Test User:/export/home/hpcuser1:/bin/bash
hpcusers:x:1001:
```

### Verify Home Directory

```bash
ls -la /export/home/
```

Expected:
```
drwx------. 2 hpcuser1 hpcusers 83 <date> hpcuser1
```

## 8.2 Passwordless SSH for Users

### Why NFS Makes This Easy
Root's SSH keys live in `/root/.ssh/` — local disk on each node. Keys
must be manually copied to every node with `ssh-copy-id`.

Regular users live in `/export/home/` — NFS shared. Generate keys once,
authorize once, and every node sees the same `authorized_keys` file
automatically. One operation, cluster-wide access.

### Generate Keys and Authorize (headnode only)

Log in as the user:
```bash
su - hpcuser1
```

Generate key pair and authorize:
```bash
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Fix SELinux for NFS Home Directories

**Critical step — easy to miss.**

SELinux on compute nodes blocks sshd from reading `authorized_keys`
on NFS mounts by default. Without this fix, key auth is silently rejected
and SSH falls back to password. The key is offered, the server refuses it,
no error message explains why.

Run on **all compute nodes** (as root):
```bash
setsebool -P use_nfs_home_dirs 1
```

`-P` makes the change persistent across reboots.

**Diagnosis:** If passwordless SSH fails with NFS homes, check:
```bash
ssh -v user@compute-1 hostname 2>&1 | grep -E "Offering|continue"
```
If you see `Offering public key` followed by `Authentications that can
continue` (key rejected), SELinux is the cause.

### Test Passwordless SSH

```bash
ssh hpcuser1@compute-1 hostname
ssh hpcuser1@compute-2 hostname
```

Both must return hostname with no password prompt.

## 8.3 Submit a SLURM Job as User

```bash
sbatch --wrap="hostname && whoami && date" --job-name=usertest -N 1
```

Check job completed:
```bash
sacct --format=JobID,JobName,State,ExitCode,NodeList,User
```

Check output file in home directory:
```bash
ls -la ~/
cat ~/slurm-*.out
```

Expected output file content:
```
compute-1
hpcuser1
Fri Apr 10 04:10:38 PM IST 2026
```

**What this proves:**
- SLURM accepts jobs from regular users
- Job runs on compute node as the correct user
- Output file written to NFS home — visible from headnode immediately

## 8.4 Adding More Users

For each new user, follow this pattern:

1. Choose next available UID (1002, 1003, etc.)
2. Create group with matching GID (if new group needed)
3. Create user on headnode with `-m` (creates home)
4. Create user on all compute nodes with `-M` (no home creation)
5. Set password on headnode
6. User generates their own SSH keys on first login

Template:
```bash
# On headnode
groupadd -g <GID> <groupname>
useradd -u <UID> -g <GID> -d /export/home/<username> \
  -m -s /bin/bash -c "<Full Name>" <username>
passwd <username>

# On all compute nodes
groupadd -g <GID> <groupname>
useradd -u <UID> -g <GID> -d /export/home/<username> \
  -M -s /bin/bash -c "<Full Name>" <username>
```

## 8.5 Scaling Consideration

Manual user creation across nodes doesn't scale beyond a handful of nodes.
Solutions for larger clusters:

| Scale       | Solution                              |
|-------------|---------------------------------------|
| 2-5 nodes   | Manual with fixed UIDs (this phase)   |
| 5-20 nodes  | Ansible playbook for user creation    |
| 20+ nodes   | FreeIPA / LDAP centralized directory  |
| Enterprise  | Active Directory + SSSD integration   |

FreeIPA is a planned future phase for this cluster.

## Key Files Reference

| File                          | Purpose                        |
|-------------------------------|--------------------------------|
| /etc/passwd                   | User records (local)           |
| /etc/group                    | Group records (local)          |
| /export/home/<user>/.ssh/     | SSH keys (NFS shared)          |
| /export/home/<user>/          | User home (NFS shared)         |

## Troubleshooting

### SSH key auth fails with NFS home
```bash
# Check SELinux boolean
getsebool use_nfs_home_dirs
# Fix
setsebool -P use_nfs_home_dirs 1
```

### Files show numeric UID instead of username
UID mismatch between nodes. Check:
```bash
getent passwd <username>  # run on each node
```
UIDs must match. Fix by deleting and recreating user with correct UID.

### User can't submit SLURM jobs
Check if user exists in SLURM accounting:
```bash
sacctmgr show user <username>
```
If not listed, SLURM may need the user added to accounting:
```bash
sacctmgr add user <username> account=root
```

### Home directory not visible on compute nodes
Check NFS mounts:
```bash
df -h | grep export
mount | grep export
```
If not mounted, check `/etc/fstab` and run `mount -a`.
