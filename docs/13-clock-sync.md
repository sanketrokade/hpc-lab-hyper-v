# Phase 13 — Clock Synchronization

## Why This Phase Exists

This phase was born from a failure. In an earlier attempt to set up FreeIPA,
authentication broke in confusing ways: a freshly-created user `ipauser1`
couldn't log in despite repeated correct passwords, and certificates appeared
to be expired immediately after issue. We rolled back to the `pre-freeipa`
snapshot without understanding the root cause.

The root cause was time skew.

Kerberos — the authentication protocol underneath FreeIPA — issues
time-stamped tickets valid for a narrow window (default 5 minutes). If the
clocks of the KDC and the client disagree by more than 5 minutes, every
ticket appears expired the moment it's issued. The user sees "incorrect
password." The actual password was always fine.

The certificate-expiry messages are the same root cause wearing a different
hat: FreeIPA's internal CA issues short-lived certs, and if the clock is
wrong, everything looks expired.

On Hyper-V VMs specifically, this is extra dangerous because:
- VMs pause and resume with the host; clocks drift hard during pauses
- Hyper-V has its own time synchronization service that fights with chronyd
- Rocky 9 minimal install doesn't include chrony by default
- Compute nodes are NAT'd through headnode, but their clocks drift independently

This phase establishes a proper time synchronization chain across all three
nodes before any FreeIPA work begins.

## Goal

Build a strict, hierarchical time synchronization chain:

```
Internet NTP (Cloudflare/pool.ntp.org)
            ↓
    headnode (stratum 4)
            ↓
    compute-1, compute-2 (stratum 5)
```

All cluster nodes must agree to within milliseconds, and compute nodes must
sync only to headnode (single source of truth).

## Pre-flight: Take a Snapshot

Before anything, snapshot all three VMs as a rollback point. Run on the
Hyper-V host in elevated PowerShell:

```powershell
Checkpoint-VM -Name "headnode"  -SnapshotName "mpi-working"
Checkpoint-VM -Name "compute-1" -SnapshotName "mpi-working"
Checkpoint-VM -Name "compute-2" -SnapshotName "mpi-working"

Get-VMSnapshot -VMName "headnode","compute-1","compute-2" |
    Select-Object VMName, Name, CreationTime |
    Format-Table -AutoSize
```

This `mpi-working` snapshot captures the cluster after Phase 12 (OpenMPI
verified) and before any clock sync changes.

## Step 1 — Disable Hyper-V Time Synchronization on All Guests

On the Hyper-V host:

```powershell
Disable-VMIntegrationService -VMName "headnode"  -Name "Time Synchronization"
Disable-VMIntegrationService -VMName "compute-1" -Name "Time Synchronization"
Disable-VMIntegrationService -VMName "compute-2" -Name "Time Synchronization"

Get-VMIntegrationService -VMName "headnode","compute-1","compute-2" |
    Where-Object Name -eq "Time Synchronization" |
    Select-Object VMName, Name, Enabled |
    Format-Table -AutoSize
```

Expected: `Enabled : False` on all three.

### Why

Hyper-V pushes the host's clock into the guest periodically. chronyd is
also trying to manage the clock. They fight. The guest clock jumps around
unpredictably — sometimes forward, sometimes backward. Kerberos tickets
become unreliable. We disable the Hyper-V side and let chronyd be the sole
authority. This is standard practice for any Linux VM running time-sensitive
services (Kerberos, databases, log aggregation, anything with TLS).

This change takes effect immediately for running VMs — no reboot needed.

## Step 2 — Configure chrony on Headnode

### 2.1 Confirm starting state

```bash
rpm -q chrony
systemctl status chronyd
date
timedatectl
```

In our case: chrony was not installed, and the headnode clock was about
12 hours wrong (showing Sunday 11:07 PM when reality was Monday 11:06 AM).
This is exactly the kind of skew that breaks Kerberos.

### 2.2 Install chrony

```bash
dnf install -y chrony
rpm -q chrony
ls -la /etc/chrony.conf
```

### 2.3 Back up the default config

```bash
cp /etc/chrony.conf /etc/chrony.conf.original
```

**Why:** Never delete vendor configs. The default config has comments
explaining every directive — useful reference material later. We replace
it with our own clean version, but keep the original around.

### 2.4 Write the headnode config

`/etc/chrony.conf`:

```
# Headnode chrony configuration
# Role: External NTP client + internal NTP server for compute nodes

# === Upstream time sources ===
# Use the global NTP pool. iburst speeds up initial sync.
pool pool.ntp.org iburst

# === Local clock behavior ===
# Record drift so chronyd learns this VM's clock tendencies over reboots.
driftfile /var/lib/chrony/drift

# Allow stepping (instant correction) only at startup, only if off by >1s.
# After startup, slew the clock smoothly. Stepping a running clock breaks
# Kerberos, databases, and log timestamps.
makestep 1.0 3

# Sync the hardware (RTC) clock from the system clock periodically.
rtcsync

# === Serve time to compute nodes ===
# Allow only the cluster subnet to query us for time.
allow 10.10.10.0/24

# === Logging ===
logdir /var/log/chrony
```

### Why each directive

- **`pool pool.ntp.org iburst`** — `pool` (vs `server`) means chrony picks
  multiple servers from the pool automatically and discards bad ones.
  `iburst` sends 4 packets in quick succession on startup so initial sync
  happens in seconds rather than minutes. Critical when the clock starts
  badly off.

- **`driftfile`** — Every clock has a slight rate error (gains or loses a
  few ppm per day). chrony measures this and writes it to disk, so after
  reboot it can pre-compensate instead of rediscovering the drift.

- **`makestep 1.0 3`** — Two-stage policy. The first three measurements
  are allowed to *step* the clock (jump it instantly) if off by more than
  1 second. After that, chrony only *slews* (gradually adjusts the rate).
  A clock jumping backward during normal operation breaks running services
  — file timestamps go backward, Kerberos tickets seem to come from the
  future, log analysis breaks. Stepping is only safe at startup.

- **`rtcsync`** — Pushes the synced system time back into the hardware
  clock. Without this, every reboot starts wrong again until NTP catches
  up. With this, the RTC stays close to correct.

- **`allow 10.10.10.0/24`** — Default chrony refuses to serve time to
  anyone. We explicitly allow only the cluster subnet. We do NOT allow
  192.168.30.0/28 (the NAT side) — no reason to serve time outward.

- **`logdir /var/log/chrony`** — When something goes wrong, you want logs.

### 2.5 Open NTP in the firewall (trusted zone only)

```bash
firewall-cmd --zone=trusted --add-service=ntp --permanent
firewall-cmd --reload
firewall-cmd --zone=trusted --list-services
```

**Why trusted zone only:** eth1 (cluster interface) is in trusted zone.
eth0 (NAT interface) is in public zone. We serve NTP only inward to compute
nodes, never outward. Putting it in public zone would create pointless
attack surface.

### 2.6 Start chronyd and verify

```bash
systemctl enable --now chronyd
systemctl status chronyd --no-pager
chronyc sources
chronyc tracking
date
timedatectl
```

What to look for:

- `^*` symbol next to one source in `chronyc sources` — means "current best
  source." Lock-on confirmation.
- `chronyc tracking` shows `System time` as a small offset (microseconds or
  milliseconds, not hours).
- `date` shows correct local time.
- `timedatectl` shows `System clock synchronized: yes` and
  `NTP service: active`.

## Why chrony, Not ntpd

The old `ntpd` daemon was the standard for two decades but had real problems
on virtualized and intermittently-connected systems — it assumed a stable
network and a clock that drifts smoothly. VMs violate both assumptions:
they pause, resume, and migrate.

`chrony` was written specifically to handle these cases. It syncs faster
after boot, handles large clock jumps gracefully, works on systems that
aren't always online, and is the default on RHEL 8+ / Rocky 8+. For an HPC
cluster on Hyper-V, chrony isn't just preferred — it's the right tool.
Using ntpd here would be fighting the OS.

## Step 3 — Configure chrony on Compute Nodes

### 3.1 Install chrony

On compute-1 (then repeat for compute-2):

```bash
dnf install -y chrony
cp /etc/chrony.conf /etc/chrony.conf.original
```

Compute nodes have internet through the headnode's NAT/masquerade, so
`dnf install` works directly.

### 3.2 Write the compute node config

`/etc/chrony.conf` (same on compute-1 and compute-2, only the comment
header differs):

```
# Compute-1 chrony configuration
# Role: NTP client only — syncs to headnode, serves no one

# === Upstream time source ===
# Headnode is the sole authority. iburst for fast initial sync.
server headnode iburst

# === Local clock behavior ===
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync

# === Logging ===
logdir /var/log/chrony

# Note: No 'allow' directive — this node does not serve time to anyone.
# Note: No 'pool' — we deliberately do not fall back to external NTP.
#       If headnode is down, we accept clock drift over splitting brain.
```

### Why this differs from headnode's config

- **`server headnode` instead of `pool pool.ntp.org`** — `server` is a
  single specific source; `pool` picks multiple from a DNS round-robin.
  We use `server` because we want exactly one source. `headnode` resolves
  via `/etc/hosts` to `10.10.10.1`.

- **No `allow` directive** — Compute nodes are pure clients. They have no
  business serving time to anything. Default chrony behavior (no `allow`)
  refuses all incoming time queries — exactly what we want.

- **No fallback to external NTP** — If compute nodes fall back to the
  internet pool independently, they might pick different upstream servers
  and disagree with each other by milliseconds. For a Kerberos/MPI
  cluster, that's worse than both being equally wrong. Single source of
  truth means *consistent* time across the cluster, even if that single
  source is briefly down. In production, this is solved by a second NTP
  server (backup headnode or dedicated time server) — out of scope for
  a 3-node lab.

### 3.3 Firewall — nothing to do on compute nodes

eth0 on compute nodes is already in the trusted zone (set up during
Phase 12 for OpenMPI). Trusted zone allows everything outbound and inbound,
so NTP traffic to headnode passes without rules.

### 3.4 Start chronyd and verify

```bash
systemctl enable --now chronyd
chronyc sources
chronyc tracking
date
timedatectl
```

Expected on compute nodes:
- `chronyc sources` shows exactly one row, `^* headnode`
- `chronyc tracking` shows Reference ID resolving to `headnode`
  (10.10.10.1, hex 0A0A0A01), Stratum 5
- `System time` offset in microseconds
- Clock snaps to correct local time

## Step 4 — Cluster-Wide Verification

From headnode:

```bash
echo "=== HEADNODE ===" && date +"%Y-%m-%d %H:%M:%S.%N" && \
echo "=== COMPUTE-1 ===" && ssh compute-1 'date +"%Y-%m-%d %H:%M:%S.%N"' && \
echo "=== COMPUTE-2 ===" && ssh compute-2 'date +"%Y-%m-%d %H:%M:%S.%N"'

chronyc tracking | grep -E "Reference ID|Stratum|System time"
ssh compute-1 'chronyc tracking | grep -E "Reference ID|Stratum|System time"'
ssh compute-2 'chronyc tracking | grep -E "Reference ID|Stratum|System time"'
```

Expected hierarchy:
- Headnode: stratum 4, reference is internet NTP (Cloudflare or pool),
  sub-millisecond offset
- Compute-1: stratum 5, reference is `headnode` (0A0A0A01),
  microsecond offset
- Compute-2: stratum 5, reference is `headnode` (0A0A0A01),
  microsecond offset

Wall-clock timestamps from the SSH command will span a few hundred
milliseconds because each SSH handshake takes time — that's network
latency, not clock skew. The actual clock offsets in `chronyc tracking`
are what matter.

## Verification Results from This Build

Before:
- Headnode: ~12 hours off
- Compute-1: ~5.5 hours off
- Compute-2: ~5.5 hours off (`-19794.821446 seconds`)

After:
- Headnode: 2.7 ms fast of internet NTP
- Compute-1: 49 µs fast of headnode
- Compute-2: 6 µs slow of headnode

All three nodes agree to within microseconds. Kerberos's 5-minute
tolerance is over 5 million times wider than worst-case skew. There is
no plausible scenario where time causes FreeIPA to fail now.

## Key Decisions

| # | Decision | Why |
|---|----------|-----|
| 1 | Disable Hyper-V time sync on guests | Prevents host clock from fighting chronyd |
| 2 | chrony, not ntpd | chrony is designed for VMs and intermittent connectivity; default on RHEL 8+ |
| 3 | Headnode = stratum 4 server, compute = stratum 5 clients | Matches air-gapped HPC pattern; single source of truth |
| 4 | `allow` only 10.10.10.0/24 on headnode | NTP served inward only, never outward |
| 5 | NTP firewall on trusted zone only | Public zone exposure is unnecessary attack surface |
| 6 | `makestep 1.0 3` | Step only at startup, slew thereafter — running services break on backward time jumps |
| 7 | `rtcsync` enabled | Hardware clock stays correct across reboots |
| 8 | `server` not `pool` on compute nodes | Exactly one source, no surprises |
| 9 | No external NTP fallback on compute nodes | Consistent cluster time beats independently-correct time |
| 10 | Backup `chrony.conf` before editing | Never delete vendor configs — useful reference |

## Files Touched

| Path | Node(s) | Purpose |
|------|---------|---------|
| `/etc/chrony.conf` | all 3 | NTP daemon configuration |
| `/etc/chrony.conf.original` | all 3 | Backup of vendor default |
| `/var/log/chrony/` | all 3 | Log directory (auto-created) |
| `/var/lib/chrony/drift` | all 3 | Drift file (auto-managed) |
| firewalld trusted zone | headnode | NTP service allowed inward |

## Closing Snapshot

After verification:

```powershell
Checkpoint-VM -Name "headnode"  -SnapshotName "post-clock-sync"
Checkpoint-VM -Name "compute-1" -SnapshotName "post-clock-sync"
Checkpoint-VM -Name "compute-2" -SnapshotName "post-clock-sync"
```

This snapshot is the last clean rollback point before FreeIPA work begins.

## Next Phase

Phase 14 — FreeIPA / Centralized User Management. The original failure
that motivated this phase will not recur, because the root cause has been
eliminated.
