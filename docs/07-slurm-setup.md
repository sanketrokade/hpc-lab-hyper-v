# Phase 7: SLURM Scheduler Setup

## Overview
Install and configure SLURM workload manager across all three nodes.
SLURM turns independent Linux nodes into a managed HPC cluster where
jobs are scheduled, tracked, and accounted for.

## SLURM Architecture

| Component  | Daemon     | Node      | Port | Purpose                              |
|------------|------------|-----------|------|--------------------------------------|
| Controller | slurmctld  | headnode  | 6817 | Accepts jobs, schedules, monitors    |
| Database   | slurmdbd   | headnode  | 6819 | Job accounting, user management      |
| Node Agent | slurmd     | compute-* | 6818 | Executes jobs, reports node status   |

**How they interact:**
- slurmctld receives job submissions and decides which node runs what
- slurmd reports node resources (CPUs, memory, state) to slurmctld
- slurmd receives job launch instructions from slurmctld
- slurmdbd records all job history, accounting, usage to MariaDB
- All communication authenticated by Munge

## Prerequisites
- All nodes networked and reachable by hostname
- Passwordless SSH configured (Phase 6 complete)
- EPEL repository available on all nodes

## 7.1 Munge Authentication

Munge provides cryptographic authentication between SLURM daemons.
All nodes share the same munge key — a message signed on one node
can be verified on any other node with the same key.

### Install Munge on All Nodes

**On headnode:**
```bash
dnf install -y epel-release
dnf install -y munge munge-libs
```

**On compute-1 and compute-2:**
```bash
dnf install -y epel-release
dnf install -y munge munge-libs
```

### Generate Munge Key (headnode only)

```bash
create-munge-key
ls -la /etc/munge/
```

Expected permissions:
```
-r--------. 1 munge munge 1024 <date> munge.key
```

Munge enforces strict permissions. Wrong ownership or mode = service
refuses to start.

### Start Munge on Headnode

```bash
systemctl enable --now munge
```

### Verify Local Munge

```bash
munge -n | unmunge
```

Expected: `STATUS: Success (0)`

### Distribute Key to Compute Nodes

```bash
scp /etc/munge/munge.key root@compute-1:/etc/munge/
scp /etc/munge/munge.key root@compute-2:/etc/munge/
```

On **both compute nodes**:

```bash
chown munge:munge /etc/munge/munge.key
chmod 400 /etc/munge/munge.key
systemctl enable --now munge
munge -n | unmunge
```

### Verify Cross-Node Authentication

From headnode:

```bash
munge -n | ssh compute-1 unmunge
munge -n | ssh compute-2 unmunge
```

Both must show `STATUS: Success (0)` and `ENCODE_HOST: headnode`.
This proves all nodes share the same key and can authenticate each other.

## 7.2 MariaDB Setup

slurmdbd requires a database backend to store job accounting data.

### Install and Secure MariaDB

```bash
dnf install -y mariadb-server
systemctl enable --now mariadb
mysql_secure_installation
```

Follow prompts:
- Switch to unix_socket auth: Y
- Change root password: Y
- Remove anonymous users: Y
- Disallow root login remotely: Y
- Remove test database: Y
- Reload privilege tables: Y

### Create SLURM Database and User

```bash
mysql -u root -p
```

```sql
CREATE DATABASE slurm_acct_db;
CREATE USER 'slurm'@'localhost' IDENTIFIED BY 'your_password_here';
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**Note:** Remember this password — it goes into slurmdbd.conf.

## 7.3 SLURM User

The SLURM package does not always create the slurm system user
automatically. Create it manually with a fixed UID on **all three nodes**.
The UID must be identical across all nodes — mismatched UIDs cause
file ownership and permission problems.

```bash
groupadd -g 994 slurm
useradd -r -u 994 -g 994 -d /var/lib/slurm -s /sbin/nologin \
  -c "SLURM workload manager" slurm
```

Verify:

```bash
getent passwd slurm
getent group slurm
```

Expected:
```
slurm:x:994:994:SLURM workload manager:/var/lib/slurm:/sbin/nologin
slurm:x:994:
```

## 7.4 Install SLURM Packages

**On headnode:**

```bash
dnf install -y slurm slurm-slurmctld slurm-slurmdbd
```

**On compute-1 and compute-2:**

```bash
dnf install -y slurm slurm-slurmd
```

Verify same version installed everywhere:

```bash
rpm -q slurm
```

Version must match across all nodes. Mismatch causes communication failures.

## 7.5 Create Required Directories

**On headnode:**

```bash
mkdir -p /var/spool/slurmctld
mkdir -p /var/log/slurm
chown slurm:slurm /var/spool/slurmctld
chown slurm:slurm /var/log/slurm
chmod 755 /var/spool/slurmctld
```

**On compute-1 and compute-2:**

```bash
mkdir -p /var/spool/slurmd
mkdir -p /var/log/slurm
chown slurm:slurm /var/spool/slurmd
chown slurm:slurm /var/log/slurm
chmod 755 /var/spool/slurmd
```

## 7.6 SLURM Configuration

### Get Actual Node Hardware Topology

Before writing slurm.conf, query the actual hardware:

```bash
ssh compute-1 "nproc && free -m | grep Mem"
```

**Important:** Hyper-V presents vCPUs as 2 cores × 2 threads, not
4 cores × 1 thread. SLURM validates config against actual hardware
and fails if they don't match.

Verify actual topology:

```bash
ssh compute-1 "lscpu | grep -E 'Socket|Core|Thread'"
```

### Create slurm.conf on Headnode

```bash
cat > /etc/slurm/slurm.conf << 'EOF'
# Cluster Identity
ClusterName=hpc-lab
SlurmctldHost=headnode

# Authentication
AuthType=auth/munge
CryptoType=crypto/munge

# Logging
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid

# Directories
StateSaveLocation=/var/spool/slurmctld
SlurmdSpoolDir=/var/spool/slurmd

# User
SlurmUser=slurm

# Scheduling
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

# Accounting
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=headnode
JobAcctGatherType=jobacct_gather/cgroup

# Node Recovery
ReturnToService=2

# Timeouts
SlurmctldTimeout=120
SlurmdTimeout=300
InactiveLimit=0
MinJobAge=300
KillWait=30
Waittime=0

# Node Definitions
NodeName=compute-1 NodeAddr=10.10.10.11 CPUs=4 RealMemory=3500 Sockets=1 CoresPerSocket=2 ThreadsPerCore=2 State=UNKNOWN
NodeName=compute-2 NodeAddr=10.10.10.12 CPUs=4 RealMemory=3500 Sockets=1 CoresPerSocket=2 ThreadsPerCore=2 State=UNKNOWN

# Partition Definition
PartitionName=compute Nodes=compute-1,compute-2 Default=YES MaxTime=INFINITE State=UP
EOF
```

**Why ReturnToService=2?**
After a reboot or slurmd restart, nodes automatically return to service
when they register with slurmctld. Without this, nodes remain DOWN
after every restart and require manual `scontrol update State=RESUME`.

**Why CoresPerSocket=2 ThreadsPerCore=2 and not CoresPerSocket=4?**
Hyper-V presents vCPUs as 2 physical cores with hyperthreading (2 threads
each). SLURM reads actual hardware topology and refuses to run if config
doesn't match. Always verify with `lscpu` before writing node definitions.

### Create slurmdbd.conf on Headnode

```bash
cat > /etc/slurm/slurmdbd.conf << 'EOF'
AuthType=auth/munge
DbdHost=headnode
DbdPort=6819
SlurmUser=slurm
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageHost=localhost
StorageUser=slurm
StoragePass=your_password_here
StorageLoc=slurm_acct_db
EOF

chmod 600 /etc/slurm/slurmdbd.conf
chown slurm:slurm /etc/slurm/slurmdbd.conf
```

`slurmdbd.conf` contains the database password. 600 owned by slurm —
readable only by the slurm user.

### Distribute slurm.conf to Compute Nodes

slurm.conf must be **identical** on all nodes. Any difference causes
communication failures or node rejection.

```bash
scp /etc/slurm/slurm.conf root@compute-1:/etc/slurm/
scp /etc/slurm/slurm.conf root@compute-2:/etc/slurm/
```

Verify identical files using md5sum:

```bash
md5sum /etc/slurm/slurm.conf
ssh compute-1 "md5sum /etc/slurm/slurm.conf"
ssh compute-2 "md5sum /etc/slurm/slurm.conf"
```

All three hashes must match exactly.

## 7.7 Firewall Configuration

SLURM daemons communicate on specific ports. These must be open.

### Port Reference

| Port | Protocol | Daemon    | Open on   |
|------|----------|-----------|-----------|
| 6817 | TCP      | slurmctld | headnode  |
| 6818 | TCP      | slurmd    | compute-* |
| 6819 | TCP      | slurmdbd  | headnode  |

**Common mistake:** Opening 6817 on compute nodes instead of 6818.
slurmd listens on 6818, not 6817.

### Headnode Firewall

```bash
firewall-cmd --zone=trusted --add-port=6817/tcp --permanent
firewall-cmd --zone=trusted --add-port=6818/tcp --permanent
firewall-cmd --zone=trusted --add-port=6819/tcp --permanent
firewall-cmd --reload
```

### Compute Node Firewall

```bash
ssh compute-1 "firewall-cmd --add-port=6818/tcp --permanent && firewall-cmd --reload"
ssh compute-2 "firewall-cmd --add-port=6818/tcp --permanent && firewall-cmd --reload"
```

### Verify Ports Open

```bash
# From headnode, test compute node ports
bash -c "echo > /dev/tcp/compute-1/6818" && echo "open" || echo "closed"
bash -c "echo > /dev/tcp/compute-2/6818" && echo "open" || echo "closed"
```

## 7.8 Start SLURM Services

Order matters — slurmdbd must be running before slurmctld connects to it.

**On headnode:**

```bash
systemctl enable --now slurmdbd
sleep 5
systemctl enable --now slurmctld
```

**On compute-1 and compute-2:**

```bash
systemctl enable --now slurmd
```

### Verify All Services

```bash
# On headnode
systemctl status slurmdbd --no-pager
systemctl status slurmctld --no-pager

# On compute nodes
ssh compute-1 "systemctl status slurmd --no-pager"
ssh compute-2 "systemctl status slurmd --no-pager"
```

### Check Cluster Status

```bash
sinfo
squeue
```

Expected `sinfo` output:
```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
compute*     up   infinite      2   idle compute-[1-2]
```

Both nodes must show `idle` (no asterisk). `idle*` means nodes haven't
fully registered yet.

## 7.9 Submit Test Jobs

```bash
sbatch --wrap="hostname && date" --job-name=test1 -N 1
sbatch --wrap="hostname && date" --job-name=test2 -N 1
sleep 15
sacct --format=JobID,JobName,State,ExitCode,NodeList
```

Expected output:
```
JobID    JobName   State     ExitCode  NodeList
-------- --------- --------- --------- ---------
5        test1     COMPLETED 0:0       compute-1
5.batch  batch     COMPLETED 0:0       compute-1
6        test2     COMPLETED 0:0       compute-2
6.batch  batch     COMPLETED 0:0       compute-2
```

Verify job output files on compute nodes:

```bash
ssh compute-1 "cat /root/slurm-5.out"
ssh compute-2 "cat /root/slurm-6.out"
```

Expected:
```
compute-1
Tue Apr  7 01:14:37 PM IST 2026
```

## Troubleshooting

### Nodes stuck in idle* state
Nodes registered but not completing handshake. Restart slurmd on
compute nodes after slurmctld is fully up:
```bash
ssh compute-1 "systemctl restart slurmd"
ssh compute-2 "systemctl restart slurmd"
```

### NODE_FAIL on job submission
Check slurmd log on the failing node:
```bash
ssh compute-1 "tail -30 /var/log/slurm/slurmd.log"
```
Common causes:
- CPU topology mismatch (CoresPerSocket/ThreadsPerCore wrong)
- Port 6818 not open on compute node
- slurmd not actually listening (check `ss -tlnp | grep slurmd`)

### Nodes not responding, setting DOWN
slurmctld can't reach slurmd. Verify port 6818 is open and listening:
```bash
bash -c "echo > /dev/tcp/compute-1/6818" && echo "open" || echo "closed"
ssh compute-1 "ss -tlnp | grep slurmd"
```

### Nodes stuck in DOWN after reboot
Without `ReturnToService=2`, nodes stay DOWN after restart. Add to
slurm.conf and redistribute to all nodes. Then restart all daemons.

### Jobs stuck in CG (completing) state
Old corrupted job state. Force cancel and clear slurmctld state:
```bash
scancel <jobid>
systemctl stop slurmctld
rm -rf /var/spool/slurmctld/*
systemctl start slurmctld
ssh compute-1 "systemctl restart slurmd"
ssh compute-2 "systemctl restart slurmd"
```

### slurmdbd failing to connect to MySQL (error 1045)
Password mismatch between slurmdbd.conf and the MySQL user.
Test the database credentials directly:
```bash
mysql -u slurm -p'yourpassword' slurm_acct_db
```
If it fails, update the MySQL user password:
```sql
ALTER USER 'slurm'@'localhost' IDENTIFIED BY 'newpassword';
FLUSH PRIVILEGES;
```
Then update StoragePass in slurmdbd.conf to match.

## Final Cluster State After Phase 7

```
sinfo output:
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
compute*     up   infinite      2   idle compute-[1-2]
```

| Service    | Node      | Status  |
|------------|-----------|---------|
| munge      | all nodes | running |
| mariadb    | headnode  | running |
| slurmdbd   | headnode  | running |
| slurmctld  | headnode  | running |
| slurmd     | compute-* | running |

Jobs submit, run on compute nodes, and complete successfully.
Job accounting stored in MariaDB via slurmdbd.
