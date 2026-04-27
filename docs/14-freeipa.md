# Phase 14 — FreeIPA / Centralized User Management

## Why This Phase Exists

Up to Phase 13, every user on the cluster was a *local* user — created
separately on each node with `useradd`, with carefully-pinned UIDs to keep
NFS file ownership consistent. SSH between nodes worked because we manually
distributed public keys and managed `authorized_keys`. This works for three
nodes. It does not scale.

Phase 14 replaces this entirely with **FreeIPA**: a unified identity and
authentication system. After this phase, the cluster has:

- One source of truth for users and groups (LDAP)
- Single sign-on across all nodes (Kerberos)
- Centralized DNS (BIND, integrated with the realm)
- Host-level identity for every node (host keytabs)
- A Certificate Authority for internal TLS

This phase also resolves the failure that motivated Phase 13 (clock
synchronization). Time skew was the root cause of an earlier broken FreeIPA
attempt; with chrony established cluster-wide, the install proceeded
cleanly.

## Goal

Build a fully-functional FreeIPA realm `HPCLAB.INTERNAL` with:

- Headnode as the IPA server (LDAP, KDC, BIND, CA, web UI)
- Compute nodes enrolled as IPA clients
- Existing local users (`hpcuser1`, `hpcuser2`) recreated as IPA users
- All cluster nodes resolving identity, DNS, and Kerberos through IPA
- Verified end-to-end: SSH (Kerberos), SLURM, OpenMPI all working as IPA
  users with no manual SSH key distribution

## Pre-Approved Architectural Decisions

Before any commands, these decisions were settled:

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| 1 | DNS strategy | Integrated DNS (FreeIPA-managed BIND) | Production HPC pattern; service discovery via SRV records works automatically |
| 2 | Domain | `hpclab.internal` | `.internal` reserved by ICANN/IANA for private networks (RFC 8375); cannot collide with public domains |
| 3 | Realm | `HPCLAB.INTERNAL` | Uppercase by convention; matches domain string |
| 4 | Hostnames | Full FQDN (`headnode.hpclab.internal` etc.) | IPA refuses to install with short hostnames; production standard |
| 5 | Existing users | Delete and recreate in IPA | Test users, no real work to migrate; avoids UID collisions with IPA's 536000000+ range |
| 6 | Implementation | Manual first; flag Ansible opportunities | Learning before automation; can't write a good playbook for something not understood at shell level |

## Step 1 — Pre-flight Verification

Before changing anything, verify state matches expectations:

```bash
chronyc tracking | grep -E "Reference ID|Stratum|System time"
hostname
hostname -f
cat /etc/hosts
ip -4 addr show
df -h /var /usr
free -h
ss -tlnp | grep -E ':(53|88|389|636|464|80|443|749)\s'
```

What we're checking:

- **Time sync** — Phase 13 must hold; stratum 4, microsecond offset
- **Hostnames** — currently short (`headnode`); will change to FQDN
- **Disk** — at least 2GB free for IPA install + data
- **Memory** — IPA's 389-ds wants ~500MB; verify headnode has it
- **Port conflicts** — nothing already listening on 53, 88, 389, 636, 464,
  80, 443, 749. The port 53 check is critical: if `systemd-resolved` is
  bound there, BIND install will fail.

## Step 2 — Hostname Migration to FQDN

FreeIPA requires fully-qualified domain names. Short hostnames cause install
failures and break service discovery.

### 2.1 Set hostname on each node

```bash
# headnode
hostnamectl set-hostname headnode.hpclab.internal

# compute-1
hostnamectl set-hostname compute-1.hpclab.internal

# compute-2
hostnamectl set-hostname compute-2.hpclab.internal
```

**Why `hostnamectl`, not editing `/etc/hostname` directly:**
`hostnamectl` writes the file, updates the kernel, and notifies systemd
atomically. Editing the file alone leaves running services with the old
name until reboot.

### 2.2 Update `/etc/hosts` on all three nodes

Backup first:

```bash
cp /etc/hosts /etc/hosts.pre-ipa
```

New content (identical on all three nodes):

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.10.10.1   headnode.hpclab.internal headnode
10.10.10.11  compute-1.hpclab.internal compute-1
10.10.10.12  compute-2.hpclab.internal compute-2
```

**Why FQDN first, short name as alias:**
The first name on a `/etc/hosts` line is the *canonical* name returned by
reverse lookups. IPA expects reverse lookups to return FQDN. Short names
remain as aliases so existing scripts still work.

### 2.3 Verify resolution

```bash
getent hosts headnode.hpclab.internal
getent hosts headnode
getent hosts 10.10.10.1
hostname -f
```

All should return the FQDN as canonical.

### 2.4 Verify SLURM still functional

SLURM doesn't need a restart — it resolves via `/etc/hosts` where short
names remain aliases:

```bash
systemctl status slurmctld --no-pager | head -10
sinfo
```

`sinfo` should still show both compute nodes in `idle` state.

## Step 3 — Pre-seed SSH known_hosts for FQDN

After hostname change, SSH known_hosts contains entries for short names but
not FQDNs. Without seeding, chained SSH (e.g., MPI launches via SSH from
compute-1 to compute-2) will fail with `Host key verification failed`.

From headnode:

```bash
# Each node knowing every node's FQDN identity
ssh compute-1 'ssh-keyscan -H compute-2.hpclab.internal >> ~/.ssh/known_hosts 2>/dev/null'
ssh compute-1 'ssh-keyscan -H compute-1.hpclab.internal >> ~/.ssh/known_hosts 2>/dev/null'
ssh compute-1 'ssh-keyscan -H headnode.hpclab.internal  >> ~/.ssh/known_hosts 2>/dev/null'

ssh compute-2 'ssh-keyscan -H compute-1.hpclab.internal >> ~/.ssh/known_hosts 2>/dev/null'
ssh compute-2 'ssh-keyscan -H compute-2.hpclab.internal >> ~/.ssh/known_hosts 2>/dev/null'
ssh compute-2 'ssh-keyscan -H headnode.hpclab.internal  >> ~/.ssh/known_hosts 2>/dev/null'

ssh-keyscan -H headnode.hpclab.internal >> ~/.ssh/known_hosts 2>/dev/null
```

**Why `-H` flag:** hashes the hostname in the file. Defensive habit — if
known_hosts is leaked, the attacker can't easily enumerate trusted hosts.

### 3.1 Fix self-SSH on compute nodes

Each compute node needs to SSH to itself for some MPI launch patterns.
Each node's `authorized_keys` only trusts other nodes — not itself.

```bash
ssh compute-1 'cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys'
ssh compute-2 'cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys'
```

### 3.2 Verify full mesh

```bash
for src in headnode compute-1 compute-2; do
  for dst in headnode.hpclab.internal compute-1.hpclab.internal compute-2.hpclab.internal; do
    if [ "$src" = "headnode" ]; then
      result=$(ssh -o BatchMode=yes $dst hostname -f 2>&1)
    else
      result=$(ssh -o BatchMode=yes $src "ssh -o BatchMode=yes $dst hostname -f" 2>&1)
    fi
    echo "$src -> $dst : $result"
  done
done
```

All 9 paths must return the destination's FQDN. `BatchMode=yes` ensures
failures are loud (no password prompts).

## Step 4 — Latent Bug: /export/scratch Permissions

While testing as `hpcuser1`, this surfaced:

```
/usr/bin/ld: cannot open output file /export/scratch/mpi-hello: Permission denied
```

`/export/scratch` was `drwxr-xr-x root:root` (mode 755) — only root could
write. This bug existed since Phase 4 but was hidden because earlier MPI
tests built binaries in user homes.

### Fix

```bash
chmod 1777 /export/scratch
ls -ld /export/scratch
```

**Why mode 1777:**
- `777` — read/write/execute for owner, group, and everyone
- `1` — the **sticky bit**: files in this directory can only be deleted by
  their owner (or root)

Same pattern as `/tmp` everywhere on Linux. Multiple users sharing a
directory need to write freely without clobbering each other's files.

`ls -ld` should show `drwxrwxrwt` — final `t` confirms sticky bit.

NFS-side note: POSIX permissions are stored on the underlying filesystem
and are visible to NFS clients immediately. No remount needed.

## Step 5 — FreeIPA Server Installation

### 5.1 Install packages

In Rocky 9, FreeIPA is shipped as regular packages in AppStream (NOT as a
DNF module — that was a Rocky 8 / RHEL 8 pattern):

```bash
dnf install -y ipa-server ipa-server-dns
rpm -q ipa-server ipa-server-dns
which ipa-server-install
```

Pulls 100+ packages: `ipa-server`, `389-ds-base`, `krb5-server`, `bind`,
`certmonger`, `httpd`, `pki-tomcatd`, and dependencies.

### 5.2 Don't start any services manually

After install, services like `dirsrv`, `named`, `krb5kdc` are inert. Do NOT
start them with `systemctl start` — `ipa-server-install` orchestrates all
of them with proper keytabs, certificates, and schemas. Starting any of
them with default config will produce a broken setup.

### 5.3 Run `ipa-server-install`

Set passwords as environment variables to keep them out of shell history:

```bash
read -s DM_PASS; echo
read -s ADMIN_PASS; echo
echo "DM_PASS length: ${#DM_PASS}"
echo "ADMIN_PASS length: ${#ADMIN_PASS}"
```

Then:

```bash
ipa-server-install \
  --realm=HPCLAB.INTERNAL \
  --domain=hpclab.internal \
  --hostname=headnode.hpclab.internal \
  --ip-address=10.10.10.1 \
  --setup-dns \
  --forwarder=8.8.8.8 \
  --forwarder=1.1.1.1 \
  --no-reverse \
  --ds-password="$DM_PASS" \
  --admin-password="$ADMIN_PASS" \
  --unattended
```

### Flag explanations

| Flag | Purpose |
|------|---------|
| `--realm=HPCLAB.INTERNAL` | Kerberos realm (uppercase by convention) — permanent |
| `--domain=hpclab.internal` | DNS domain (lowercase) — matches realm |
| `--hostname=headnode.hpclab.internal` | Must match system hostname (Step 2) |
| `--ip-address=10.10.10.1` | IP IPA advertises in DNS (cluster network only — not NAT side) |
| `--setup-dns` | Install and configure BIND |
| `--forwarder=8.8.8.8 --forwarder=1.1.1.1` | Where BIND forwards external queries; two for redundancy |
| `--no-reverse` | Skip reverse zone creation (added later in Step 7 deliberately) |
| `--ds-password=` | 389-ds Directory Manager password — used rarely (LDAP recovery) |
| `--admin-password=` | IPA `admin` user password — used constantly |
| `--unattended` | Fail fast; no interactive prompts |

### Install duration and stages

5–10 minutes. The installer:

1. Configures NTP (detects existing chrony, leaves Phase 13 config intact)
2. Generates self-signed CA, issues itself certificates from it
3. Installs and configures 389-ds (LDAP)
4. Configures Kerberos KDC and kadmin
5. Configures BIND with LDAP backend
6. Configures Apache httpd for the web UI
7. Bootstraps the IPA admin account
8. Auto-enrolls headnode as a client (configures SSSD, publishes SSH host
   keys, sets up `/etc/krb5.conf`, etc.)

### What success looks like

```
The ipa-client-install command was successful
==============================================================================
Setup complete

Next steps:
        1. You must make sure these network ports are open: ...
        2. You can now obtain a kerberos ticket using the command: 'kinit admin'

The ipa-server-install command was successful
```

## Step 6 — Server Verification

```bash
ipactl status
kinit admin   # prompts for $ADMIN_PASS
klist
ipa user-find
ipa host-find
ipa config-show
dig @10.10.10.1 headnode.hpclab.internal +short
dig @10.10.10.1 _kerberos._tcp.hpclab.internal SRV +short
dig @10.10.10.1 google.com +short
curl -k -s -o /dev/null -w "%{http_code}\n" https://headnode.hpclab.internal/ipa/ui/
```

Expected:
- `ipactl status` shows all 9 services RUNNING (Directory Service, krb5kdc,
  kadmin, named, httpd, ipa-custodia, pki-tomcatd, ipa-otpd,
  ipa-dnskeysyncd)
- `kinit admin` succeeds silently; `klist` shows TGT for
  `admin@HPCLAB.INTERNAL`
- `ipa user-find` shows 1 user (admin, UID 536000000)
- `ipa host-find` shows 1 host (headnode, with SSH fingerprints in LDAP)
- DNS forward, SRV, and forwarding queries all return correct answers
- Web UI returns HTTP 200

## Step 7 — Reconcile Post-Install Changes

The installer might have modified existing config. In our case, IPA was
respectful:

### 7.1 chrony

```bash
diff /etc/chrony.conf.original /etc/chrony.conf
chronyc sources
```

IPA detected our existing chrony with appropriate `allow` directive and
left it alone. No reconciliation needed.

### 7.2 firewalld

```bash
firewall-cmd --get-default-zone
firewall-cmd --zone=public --list-all
firewall-cmd --zone=trusted --list-all
```

Public zone (eth0, NAT-side) was unchanged: `cockpit, dhcpv6-client, ssh`
plus our existing Grafana (3000) and Prometheus (9090). **No IPA ports
exposed on NAT side.**

Trusted zone (eth1, cluster-side) has `target: ACCEPT`, so IPA's ports
(53, 88, 389, 636, 464, 80, 443) are accessible to cluster nodes through
the catch-all without explicit service rules.

This security posture is correct: IPA is reachable on cluster network,
not on NAT.

### 7.3 Note on bind addresses

IPA daemons bind to wildcard `0.0.0.0` for most services (LDAP, Kerberos,
HTTP). The firewall is what enforces the boundary, not the bind address.
Public zone simply doesn't allow these ports. If firewall rules ever
change, this assumption needs reverification.

## Step 8 — Add IPA DNS Records for Compute Nodes

The server install with `--no-reverse` left two issues that show up during
client enrollment:

```
Hostname (compute-1.hpclab.internal) does not have A/AAAA record.
Missing reverse record(s) for address(es): 10.10.10.11.
```

Cluster nodes resolve via `/etc/hosts` only — IPA's DNS doesn't know about
them. We fix this BEFORE enrolling clients to keep DNS as the source of
truth.

### 8.1 Create reverse zone for cluster network

```bash
ipa dnszone-add 10.10.10.in-addr.arpa.
```

### 8.2 Add A records for all cluster nodes

```bash
ipa dnsrecord-add hpclab.internal compute-1 --a-rec=10.10.10.11
ipa dnsrecord-add hpclab.internal compute-2 --a-rec=10.10.10.12
```

(headnode's A record was created automatically by the server install.)

### 8.3 Add PTR records for reverse resolution

```bash
ipa dnsrecord-add 10.10.10.in-addr.arpa. 1  --ptr-rec=headnode.hpclab.internal.
ipa dnsrecord-add 10.10.10.in-addr.arpa. 11 --ptr-rec=compute-1.hpclab.internal.
ipa dnsrecord-add 10.10.10.in-addr.arpa. 12 --ptr-rec=compute-2.hpclab.internal.
```

**The trailing dot on `--ptr-rec` values matters.** Without it, BIND would
append the zone name and create nonsense.

### 8.4 Verify

```bash
dig @10.10.10.1 compute-1.hpclab.internal +short
dig @10.10.10.1 -x 10.10.10.11 +short
ipa dnsrecord-find hpclab.internal
ipa dnsrecord-find 10.10.10.in-addr.arpa.
```

Forward and reverse should both work for all three nodes.

## Step 9 — Enroll Compute Nodes

### 9.1 Switch DNS to point at IPA's BIND

On each compute node:

```bash
nmcli con mod cluster ipv4.dns "10.10.10.1 8.8.8.8"
nmcli con mod cluster ipv4.dns-search "hpclab.internal"
nmcli con down cluster && nmcli con up cluster
cat /etc/resolv.conf
```

**Why two DNS servers in this order:**
- `10.10.10.1` first — IPA's BIND, knows everything about
  `hpclab.internal` and forwards external queries
- `8.8.8.8` second — pure fallback for external resolution if headnode is
  down. Cannot resolve internal names but keeps the box functional for
  package updates.

**Why `dns-search hpclab.internal`:** lets `headnode` (no dot) auto-resolve
to `headnode.hpclab.internal`.

### 9.2 Install IPA client package

```bash
dnf install -y ipa-client
```

This pulls `bind-utils` (provides `dig`) as a dependency.

### 9.3 Run `ipa-client-install`

```bash
read -s ADMIN_PASS; echo

ipa-client-install \
  --domain=hpclab.internal \
  --realm=HPCLAB.INTERNAL \
  --server=headnode.hpclab.internal \
  --hostname=compute-1.hpclab.internal \
  --principal=admin \
  --password="$ADMIN_PASS" \
  --mkhomedir \
  --no-ntp \
  --unattended
```

(Replace `compute-1` with `compute-2` for the second node.)

### Flag explanations

| Flag | Purpose |
|------|---------|
| `--domain` / `--realm` | Match server; could be auto-discovered but explicit is safer |
| `--server=headnode.hpclab.internal` | Which IPA server to enroll against |
| `--hostname=compute-N.hpclab.internal` | Must match system hostname |
| `--principal=admin` | Who we authenticate as to perform enrollment |
| `--password=...` | The admin password |
| `--mkhomedir` | Configure PAM to auto-create `/home/<user>` on first login |
| `--no-ntp` | Don't reconfigure chrony (Phase 13 setup is intentional) |
| `--unattended` | Fail fast; no prompts |

### What happens during enrollment

The client install:

1. Discovers IPA via DNS SRV records
2. Authenticates as admin to create a host principal
3. Receives a host keytab — `/etc/krb5.keytab` — the node's machine
   identity in Kerberos
4. Configures `/etc/krb5.conf`, `/etc/sssd/sssd.conf`, `/etc/openldap/ldap.conf`
5. Configures PAM for SSSD authentication
6. Configures sshd for GSSAPI authentication
7. Updates `/etc/nsswitch.conf` so `getent passwd` queries SSSD
8. Publishes the node's SSH host keys to LDAP

Takes 1–2 minutes per node.

### 9.4 Verify enrollment

After both nodes are enrolled:

```bash
ipa host-find
```

Should show 3 hosts (headnode + both computes), each with all three SSH
host key fingerprints.

## Step 10 — User Management Migration

### 10.1 Backup home directories

```bash
mkdir -p /root/pre-ipa-user-backup
cp -a /export/home/hpcuser1 /root/pre-ipa-user-backup/
cp -a /export/home/hpcuser2 /root/pre-ipa-user-backup/
```

`-a` preserves ownership, perms, timestamps. Even if we never reference
the backup, discipline matters.

### 10.2 Kill any leftover user processes

`userdel` refuses to remove a user with active processes:

```bash
ps -u hpcuser1 -u hpcuser2
pkill -u hpcuser1
pkill -u hpcuser2
sleep 2
```

If `pkill` doesn't clear them, escalate with `pkill -9 -u <user>`.

### 10.3 Remove local users on all nodes

```bash
# headnode
userdel hpcuser1
userdel hpcuser2

# compute-1
ssh compute-1 'userdel hpcuser1; userdel hpcuser2'

# compute-2
ssh compute-2 'userdel hpcuser1; userdel hpcuser2'
```

`userdel` (no `-r`) leaves home directories alone — we manage those
separately because they're on NFS.

### 10.4 Remove home directories and group

```bash
rm -rf /export/home/hpcuser1
rm -rf /export/home/hpcuser2

groupdel hpcusers   # only on headnode if needed; compute nodes' groups
                    # are auto-removed when their last user is deleted
```

### 10.5 Verify clean state

```bash
for node in headnode compute-1 compute-2; do
  echo "=== $node ==="
  if [ "$node" = "headnode" ]; then
    getent passwd hpcuser1 hpcuser2
    getent group hpcusers
  else
    ssh $node 'getent passwd hpcuser1 hpcuser2; getent group hpcusers'
  fi
done
ls /export/home/
```

All three nodes should return absolutely nothing for those queries.

### 10.6 Create IPA group

```bash
kinit admin   # if ticket expired
ipa group-add hpcusers --desc="HPC cluster users"
```

IPA auto-assigns GID from the 536000000+ range. We do not pin specific
numbers — UID/GID consistency is now enforced by SSSD pulling from LDAP.

### 10.7 Create IPA users

```bash
ipa user-add hpcuser1 \
  --first=HPC \
  --last="User One" \
  --shell=/bin/bash \
  --homedir=/export/home/hpcuser1 \
  --random

ipa user-add hpcuser2 \
  --first=HPC \
  --last="User Two" \
  --shell=/bin/bash \
  --homedir=/export/home/hpcuser2 \
  --random
```

**Flag explanations:**

| Flag | Purpose |
|------|---------|
| `--first` / `--last` | Required by LDAP schema |
| `--shell=/bin/bash` | Override IPA's default `/bin/sh` |
| `--homedir=/export/home/<user>` | NFS home, not local `/home` |
| `--random` | Generate strong one-time password; printed once at creation |

The random password is printed in the output. Save it temporarily — needed
for first-login. IPA will require change on first use.

### 10.8 Add users to hpcusers group

```bash
ipa group-add-member hpcusers --users=hpcuser1
ipa group-add-member hpcusers --users=hpcuser2
```

By default IPA users are in `ipausers` (a default access group). Adding to
`hpcusers` makes that a *supplementary* group. Each user also has a private
primary group with the same name as their username (modern Linux pattern).

### 10.9 Verify cluster-wide visibility

```bash
for node in headnode compute-1 compute-2; do
  echo "=== $node ==="
  if [ "$node" = "headnode" ]; then
    getent passwd hpcuser1 hpcuser2
    getent group hpcusers
  else
    ssh $node 'getent passwd hpcuser1 hpcuser2; getent group hpcusers'
  fi
done
```

All three nodes should return identical entries — same UIDs, same homedir,
same shell, same group membership. **Zero file editing needed on compute
nodes.** SSSD queries IPA's LDAP via Kerberos and identity propagates
automatically. This is the moment centralized identity is proven working.

## Step 11 — First-Login Test and Diagnosis

### 11.1 First login attempt

```bash
ssh hpcuser1@headnode.hpclab.internal
```

Use the random password from Step 10.7. IPA will require an immediate
password change. After change, the user is dropped at a shell prompt.

### 11.2 Issue: home directory not created on headnode

On the first login attempt, headnode produced:

```
Could not chdir to home directory /export/home/hpcuser1: No such file or directory
[hpcuser1@headnode /]$
```

Authentication succeeded — Kerberos TGT was issued — but
`pam_oddjob_mkhomedir.so` did not fire to create the home directory.

### 11.3 Diagnosis

When `ipa-server-install` auto-enrolled headnode as a client, it did NOT
pass `--mkhomedir`. We passed that flag only to `ipa-client-install` on
compute nodes. Compute nodes had `with-mkhomedir` in their authselect
profile; headnode did not.

```bash
authselect current
# headnode: Profile ID: sssd, with-sudo
# compute-1: Profile ID: sssd, with-mkhomedir, with-sudo
# compute-2: Profile ID: sssd, with-mkhomedir, with-sudo
```

### 11.4 Fix

```bash
authselect select sssd with-mkhomedir with-sudo --force
systemctl enable --now oddjobd
```

`authselect select ... --force` regenerates `/etc/pam.d/` files
(symlinks to `/etc/authselect/*`) with the new feature applied. `--force`
is required because the profile is already selected.

### 11.5 Known issue (deferred)

After authselect fix, `pam_oddjob_mkhomedir.so` is correctly present in
`/etc/authselect/system-auth` and `password-auth`, oddjobd is running, the
NFS export has `no_root_squash`, and SELinux shows no AVC denials — but
`mkhomedir` still does not fire on headnode logins. compute-1 and
compute-2 both create homes correctly under identical configuration.

In practice this is non-blocking: when a user first logs into compute-1
or compute-2, the home is created on the shared NFS filesystem, and
subsequent headnode logins find it cleanly. The headnode-specific PAM
oddity is documented for future investigation during security hardening
(Phase 15).

### 11.6 Verify login works after home exists

```bash
ssh hpcuser1@headnode.hpclab.internal
pwd        # /export/home/hpcuser1
klist      # shows TGT auto-acquired by PAM during login
id         # shows UID 536000004, both primary and supplementary groups
```

`klist` showing a TGT acquired without manual `kinit` proves end-to-end
Kerberos SSO works. Last time's failure mode (right password rejected)
is officially gone.

## Step 12 — SSSD Cache Refresh

After cleaning up local users and creating IPA users with the same names,
SSSD's negative cache may still hold stale entries. Symptoms: `id <user>`
shows only primary group, supplementary groups missing.

```bash
sss_cache -E
systemctl restart sssd
sleep 3
ssh compute-1 'sss_cache -E && systemctl restart sssd'
ssh compute-2 'sss_cache -E && systemctl restart sssd'
sleep 3
```

`sss_cache -E` invalidates every cache entry. `systemctl restart sssd`
clears in-memory state. After this, `id hpcuser1` correctly shows both
the primary group (`hpcuser1`) and supplementary `hpcusers`.

## Step 13 — End-to-End Functional Tests

### 13.1 Compute-to-compute SSH via Kerberos

```bash
ssh hpcuser1@compute-1
klist
ssh compute-2 hostname
klist
```

After `ssh compute-2` succeeds **without password prompt**, `klist` shows
a new entry: `host/compute-2.hpclab.internal@HPCLAB.INTERNAL`. That's a
Kerberos service ticket SSH acquired automatically. **No SSH keys
involved** — GSSAPI replaces the SSH-key infrastructure for users.

This is enabled by `/etc/ssh/sshd_config.d/04-ipa.conf` which IPA installs
during enrollment. It sets `GSSAPIAuthentication yes` server-side and
client-side defaults match.

### 13.2 SLURM job as IPA user

```bash
ssh hpcuser1@headnode.hpclab.internal
cd ~
sbatch --wrap="hostname; id; date" --nodes=2 --ntasks=2 --partition=compute --output=ipa-job-%j.out
sleep 5
sacct -X --format=JobID,JobName,User,State,ExitCode -p | tail -3
cat ipa-job-*.out
```

Expected: `State=COMPLETED`, `ExitCode=0:0`, output shows the IPA UID
(536000004) and `hpcusers` group membership.

SLURM uses the system's UID resolution at job execution time — it accepts
IPA users with no modification.

### 13.3 Multi-node MPI as IPA user

```bash
module load openmpi/4.1.8

cat > ~/mpi-ipa-test.c << 'EOF'
#include <mpi.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    char hostname[256];
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    gethostname(hostname, 256);
    printf("Rank %d of %d on %s as UID %d\n",
           rank, size, hostname, getuid());
    MPI_Finalize();
    return 0;
}
EOF

mpicc ~/mpi-ipa-test.c -o /export/scratch/mpi-ipa-test

mpirun --mca plm rsh \
       --prefix /export/apps/openmpi/4.1.8 \
       -np 4 \
       --host compute-1:2,compute-2:2 \
       /export/scratch/mpi-ipa-test
```

Expected: 4 lines, "Rank N of 4 on compute-X.hpclab.internal as UID
536000004", with both compute hostnames present.

**No SSH keys, no munge tricks, no host file edits.** Kerberos handles
authentication between compute nodes entirely. This is the proof that
centralized identity replaces hand-managed SSH trust for HPC workloads.

The harmless `librdmacm.so.1` warnings in the output are because OpenMPI
was built with InfiniBand support but no IB hardware exists in this lab.
OpenMPI logs and ignores them, falls back to TCP.

## Key Decisions

| # | Decision | Why |
|---|----------|-----|
| 1 | Realm `HPCLAB.INTERNAL` | RFC 8375 `.internal` reserved for private use; no public collision |
| 2 | FQDN hostnames | IPA hard requirement; production standard |
| 3 | `/etc/hosts` keeps short-name aliases | Backwards compatibility with existing scripts |
| 4 | Pre-seed FQDN known_hosts before IPA | Chained SSH (MPI launches) needs FQDN host keys |
| 5 | Self-SSH fix on compute nodes | Some MPI launch patterns require self-SSH |
| 6 | `--no-reverse` at server install | Add reverse zone deliberately in Step 8 |
| 7 | `--ip-address=10.10.10.1` (cluster IP, not NAT) | IPA serves identity to cluster, not NAT path |
| 8 | `--no-ntp` on client install | Phase 13 chrony setup is intentional, don't disturb |
| 9 | Add A/PTR records to IPA before enrolling clients | Avoids enrollment-time warnings; DNS as source of truth |
| 10 | Recreate users in IPA, don't migrate | Test users; clean break; avoids UID-range collisions |
| 11 | `--random` passwords for new users | Strong by default; one-time use; forces change on login |
| 12 | NFS home (`/export/home/<user>`) for HPC users | Matches existing storage layout |
| 13 | `admin` keeps local home (`/home/admin`) | Emergency-fix scenarios — admin should work even if NFS is down |
| 14 | mode 1777 on `/export/scratch` | Sticky bit for shared user write space (matches `/tmp` pattern) |
| 15 | `sss_cache -E` after user changes | Force fresh LDAP queries; eliminates stale negative-cache entries |

## Known Issues (Deferred)

| Issue | Impact | Notes |
|-------|--------|-------|
| `mkhomedir` does not fire on headnode logins | First login on headnode lands at `/`; subsequent logins work after compute node creates home on NFS | All preconditions appear correct (PAM stack, oddjobd, NFS perms, SELinux). Defer to Phase 15. |
| No reverse zone for `192.168.30.0/28` | NAT-side reverse lookups fail | Not currently needed; add if future tooling requires it |
| `ipausers` group has no GID | Not visible to `getent group ipausers` | IPA design choice for default access group; harmless |

## Files Touched

| Path | Node(s) | Purpose |
|------|---------|---------|
| `/etc/hostname` | all | Set FQDN |
| `/etc/hosts` | all | FQDN canonical, short alias |
| `/etc/hosts.pre-ipa` | all | Backup of pre-FQDN hosts |
| `/etc/krb5.conf` | all | Kerberos realm config |
| `/etc/sssd/sssd.conf` | all | SSSD identity provider config |
| `/etc/openldap/ldap.conf` | all | OpenLDAP client config |
| `/etc/ssh/sshd_config.d/04-ipa.conf` | all | GSSAPI auth, host key from LDAP |
| `/etc/nsswitch.conf` | all | passwd/group/etc. via SSSD |
| `/etc/krb5.keytab` | all | Host principal keytab |
| `/etc/ipa/default.conf` | all | IPA client config |
| `/root/.ssh/known_hosts` | all | Pre-seeded FQDN host keys |
| `/root/.ssh/authorized_keys` | compute-1, compute-2 | Self-SSH key added |
| `/etc/authselect/*` | all | PAM/NSS profile config |
| `/etc/pki/ca-trust/source/anchors/ipa-ca.crt` | all | IPA CA certificate (system trust) |
| `/var/lib/dirsrv/`, `/var/lib/ipa/`, `/var/named/` | headnode | IPA data directories |
| `/export/home/hpcuser1`, `/export/home/hpcuser2` | NFS | User home directories |
| `/export/scratch` | NFS | Mode changed to 1777 |
| `/root/pre-ipa-user-backup/` | headnode | Pre-deletion home backup |

## Service Inventory After Phase 14

| Service | Node | Port | Notes |
|---------|------|------|-------|
| dirsrv (389-ds) | headnode | 389/tcp, 636/tcp | LDAP backend |
| krb5kdc | headnode | 88/tcp, 88/udp | Kerberos KDC |
| kadmin | headnode | 464/tcp, 464/udp, 749/tcp | Kerberos admin |
| named (BIND) | headnode | 53/tcp, 53/udp | Realm DNS + forwarding |
| httpd | headnode | 80/tcp, 443/tcp | Web UI |
| pki-tomcatd | headnode | 8443/tcp (internal) | Certificate Authority |
| ipa-custodia | headnode | — | Secret/key management |
| ipa-otpd | headnode | — | One-time password daemon |
| ipa-dnskeysyncd | headnode | — | DNSSEC key sync |
| oddjobd | all | — | mkhomedir helper |
| sssd | all | — | Identity / auth client |

## Closing Snapshot

```powershell
Checkpoint-VM -Name "headnode"  -SnapshotName "post-freeipa"
Checkpoint-VM -Name "compute-1" -SnapshotName "post-freeipa"
Checkpoint-VM -Name "compute-2" -SnapshotName "post-freeipa"
```

Rollback layers now in place:

| Snapshot | Date | Purpose |
|----------|------|---------|
| pre-freeipa | April 16, 2026 | Deep history (failed FreeIPA attempt) |
| mpi-working | April 27, 2026 AM | Post-MPI clean state |
| post-clock-sync | April 27, 2026 PM | Pre-FreeIPA, time-synced |
| post-freeipa | April 27, 2026 PM | FreeIPA functional, end-to-end verified |

## Next Phase

Phase 15 — Security hardening. Topics include:
- HBAC (Host-Based Access Control) — restrict which users can SSH to
  which hosts via IPA
- Sudo policies via IPA — centralized sudo rules
- `ssh-trust-dns` — verify SSH host keys via DNS SSHFP records
- Investigate and resolve the headnode `mkhomedir` issue
- Firewall audit
- Sshd hardening (disable root login, key-only for admin paths)
- Audit logging
