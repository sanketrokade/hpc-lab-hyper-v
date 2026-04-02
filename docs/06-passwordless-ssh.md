# Phase 6: Passwordless SSH

## Overview
Configure passwordless SSH across all cluster nodes to enable MPI job
launching, administrative tasks, and future automation.

## Why Passwordless SSH in HPC

### MPI Job Launching
When SLURM launches an MPI job across multiple nodes, MPI uses SSH
directly to spawn processes on remote nodes. No human is in the loop.
If SSH prompts for a password, MPI fails — there's nobody to type it.

### Compute-to-Compute Communication
MPI jobs don't route all communication through the headnode. Compute
nodes talk peer-to-peer. If compute-1 can't SSH to compute-2, multi-node
MPI jobs fail.

### Two Authentication Layers in HPC
- **SSH** — process launching across nodes (MPI), admin tasks, file transfers
- **Munge** — SLURM-specific, authenticates job requests between SLURM daemons

## Prerequisites
- All three nodes running and networked (Phase 5 complete)
- /etc/hosts configured on all nodes
- All nodes reachable by hostname

## 6.1 Headnode Key Generation

Generate a 4096-bit RSA key pair with no passphrase:

```bash
ssh-keygen -t rsa -b 4096 -N "" -f /root/.ssh/id_rsa
```

**Why `-N ""`?**
`-N` sets the passphrase on the private key. Empty passphrase means no
human interaction required. If a passphrase were set, every SSH connection
would prompt for it — defeating the purpose for MPI and automation.

**Security tradeoff:** An unprotected private key is a risk if stolen. On
an isolated cluster network behind a private switch, this is standard practice.

## 6.2 Distribute Headnode Key

Copy the public key to all nodes (including headnode itself):

```bash
ssh-copy-id root@headnode
ssh-copy-id root@compute-1
ssh-copy-id root@compute-2
```

Each will prompt for the password one final time.

**Why headnode to itself?** Jobs and scripts sometimes SSH to localhost.
SLURM may also reference the headnode by hostname.

### Verify from headnode:

```bash
ssh root@headnode hostname
ssh root@compute-1 hostname
ssh root@compute-2 hostname
```

Each should return the hostname with no password prompt.

## 6.3 Compute Node Key Generation and Distribution

Compute nodes need to SSH to each other for multi-node MPI jobs.

**On compute-1:**

```bash
ssh-keygen -t rsa -b 4096 -N "" -f /root/.ssh/id_rsa
ssh-copy-id root@headnode
ssh-copy-id root@compute-2
```

**On compute-2:**

```bash
ssh-keygen -t rsa -b 4096 -N "" -f /root/.ssh/id_rsa
ssh-copy-id root@headnode
ssh-copy-id root@compute-1
```

### Verify compute-to-compute:

From compute-1:

```bash
ssh root@compute-2 hostname
```

From compute-2:

```bash
ssh root@compute-1 hostname
```

Both should return the remote hostname with no password prompt.

## 6.4 SSH Trust Matrix

| From \ To  | headnode | compute-1 | compute-2 |
|------------|----------|-----------|-----------|
| headnode   | ✓        | ✓         | ✓         |
| compute-1  | ✓        | —         | ✓         |
| compute-2  | ✓        | ✓         | —         |

All nodes can reach all other nodes without password.

## Scaling Beyond the Lab

In production with hundreds or thousands of nodes, nobody runs
`ssh-copy-id` manually. Solutions include:

1. **Shared home via NFS** — regular users' `~/.ssh/` is on shared
   storage. Generate keys once, works everywhere. Already built into
   our cluster design.
2. **Ansible** — push keys to all nodes in one playbook
3. **Image-based provisioning** — Warewulf/xCAT bake keys into the
   compute node image. Every node boots identical.
4. **Configuration management** — Puppet, Chef, Salt

What we did manually with root is the **bootstrap problem** — you need
initial access to set up the automation that prevents manual work.
Root on local disk is that bootstrap.

## Key Files Reference

| File                     | Purpose                          |
|--------------------------|----------------------------------|
| ~/.ssh/id_rsa            | Private key (never share)        |
| ~/.ssh/id_rsa.pub        | Public key (distribute freely)   |
| ~/.ssh/authorized_keys   | Public keys allowed to log in    |
| ~/.ssh/known_hosts       | Fingerprints of trusted servers  |
