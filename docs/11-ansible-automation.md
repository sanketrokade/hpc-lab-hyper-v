# Phase 11: Automation (Ansible)

## Overview
Install and configure Ansible on headnode to automate cluster operations.
Ansible eliminates repetitive manual tasks across nodes — user creation,
service checks, config distribution, package installation — replacing
multiple SSH sessions with a single command.

## Why Ansible for HPC

### The Manual Problem
Every cluster operation currently requires repeating the same commands
on every node:
- Create user → 3 SSH sessions, 3 commands, 3 verifications
- Update a config → copy to each node, restart service on each node
- Install a package → SSH to each node, run dnf, verify

For 3 nodes this is annoying. For 300 nodes it's impossible.

### Why Ansible Specifically
- **Agentless** — uses SSH which the cluster already has. Nothing to
  install on compute nodes.
- **Idempotent** — checks current state before acting. Running the same
  playbook twice changes nothing if everything is already correct. Safe
  to run repeatedly.
- **Readable** — YAML playbooks are self-documenting. A playbook IS
  the documentation.
- **Push-based** — headnode pushes changes to all nodes simultaneously.
  No daemons waiting for instructions.

### Ansible vs Shell Scripts
A shell script runs commands blindly. Ansible checks state first:
- Script: `useradd hpcuser2` → fails if user exists
- Ansible: checks if user exists → creates only if missing → reports ok

## Core Concepts

| Concept    | What it is                                           |
|------------|------------------------------------------------------|
| Inventory  | List of nodes and groups                             |
| Playbook   | YAML file describing tasks to run                    |
| Task       | Single action (create user, install package, etc.)   |
| Module     | Built-in function (user, group, systemd, copy, etc.) |
| Variable   | Reusable value (username, uid, path)                 |
| Idempotent | Same playbook, same result, no matter how many runs  |

## Prerequisites
- Passwordless SSH from headnode to all nodes (Phase 6 complete)
- All nodes reachable by hostname
- EPEL repository available on headnode

## 11.1 Install Ansible

Ansible only needs to be installed on **headnode** — the control node.
Compute nodes need nothing extra — Ansible uses existing SSH.

```bash
dnf install -y ansible
ansible --version
```

## 11.2 Project Structure

```bash
mkdir -p /etc/ansible/hpc-lab
cd /etc/ansible/hpc-lab
mkdir -p inventory group_vars host_vars roles playbooks
```

### Directory Purpose

| Directory  | Purpose                                          |
|------------|--------------------------------------------------|
| inventory  | Node lists and groups                            |
| group_vars | Variables applied to node groups                 |
| host_vars  | Variables specific to individual nodes           |
| roles      | Reusable task bundles                            |
| playbooks  | Main playbook files                              |

## 11.3 Ansible Configuration

```bash
cat > /etc/ansible/hpc-lab/ansible.cfg << 'EOF'
[defaults]
inventory = inventory/hosts
host_key_checking = False
remote_user = root
private_key_file = /root/.ssh/id_rsa

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
EOF
```

**Why host_key_checking = False?**
Nodes are known and trusted on the private cluster network. Disabling
strict host key checking prevents prompts when connecting by IP address
instead of hostname. Not appropriate for public-facing systems.

**Why ControlPersist=60s?**
Keeps SSH connections open for 60 seconds after a task completes.
Subsequent tasks reuse the connection instead of opening a new one.
Significant speedup on playbooks with many tasks.

## 11.4 Inventory

```bash
cat > /etc/ansible/hpc-lab/inventory/hosts << 'EOF'
[headnodes]
headnode ansible_host=10.10.10.1

[compute]
compute-1 ansible_host=10.10.10.11
compute-2 ansible_host=10.10.10.12

[cluster:children]
headnodes
compute

[cluster:vars]
ansible_user=root
ansible_ssh_private_key_file=/root/.ssh/id_rsa
EOF
```

**Group design:**
- `headnodes` — headnode only, for headnode-specific tasks
- `compute` — both compute nodes, for compute-specific tasks
- `cluster` — all nodes, for cluster-wide tasks

**Important:** Don't name a group the same as a host. Use `headnodes`
(plural) not `headnode` to avoid Ansible warnings.

### Verify Connectivity

```bash
cd /etc/ansible/hpc-lab
ansible -i inventory/hosts cluster -m ping
```

Expected — all three nodes return `pong`:
```
headnode  | SUCCESS => { "ping": "pong" }
compute-1 | SUCCESS => { "ping": "pong" }
compute-2 | SUCCESS => { "ping": "pong" }
```

## 11.5 Playbook: Cluster Facts

Gather and display information from all nodes simultaneously.

```bash
cat > /etc/ansible/hpc-lab/playbooks/cluster-facts.yml << 'EOF'
---
- name: Gather cluster facts
  hosts: cluster
  gather_facts: yes

  tasks:
    - name: Show hostname
      debug:
        msg: "{{ inventory_hostname }} — {{ ansible_hostname }}"

    - name: Show memory
      debug:
        msg: "RAM: {{ ansible_memtotal_mb }} MB"

    - name: Show CPU count
      debug:
        msg: "CPUs: {{ ansible_processor_vcpus }}"

    - name: Show OS
      debug:
        msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
EOF
```

Run:
```bash
ansible-playbook -i inventory/hosts playbooks/cluster-facts.yml
```

### Ansible Facts Reference
Ansible automatically collects system facts when `gather_facts: yes`.
Useful facts for HPC:

| Fact                      | Value                  |
|---------------------------|------------------------|
| ansible_hostname          | Short hostname         |
| ansible_memtotal_mb       | Total RAM in MB        |
| ansible_processor_vcpus   | Number of vCPUs        |
| ansible_distribution      | OS name (Rocky)        |
| ansible_distribution_version | OS version (9.7)   |
| ansible_default_ipv4.address | Primary IP address |

## 11.6 Playbook: Add User

Creates a cluster user correctly across all nodes — home directory on
headnode only, matching UID/GID everywhere. Handles the headnode vs
compute node difference automatically.

```bash
cat > /etc/ansible/hpc-lab/playbooks/add-user.yml << 'EOF'
---
- name: Add HPC user to all nodes
  hosts: cluster
  gather_facts: no

  vars:
    username: hpcuser2
    uid: 1002
    gid: 1002
    fullname: "HPC Test User 2"
    group: hpcusers

  tasks:
    - name: Ensure hpcusers group exists
      group:
        name: "{{ group }}"
        gid: "{{ gid }}"
        state: present

    - name: Create user on headnode with home directory
      user:
        name: "{{ username }}"
        uid: "{{ uid }}"
        group: "{{ group }}"
        home: "/export/home/{{ username }}"
        create_home: yes
        shell: /bin/bash
        comment: "{{ fullname }}"
        state: present
      when: inventory_hostname == "headnode"

    - name: Create user on compute nodes without home directory
      user:
        name: "{{ username }}"
        uid: "{{ uid }}"
        group: "{{ group }}"
        home: "/export/home/{{ username }}"
        create_home: no
        shell: /bin/bash
        comment: "{{ fullname }}"
        state: present
      when: inventory_hostname != "headnode"

    - name: Verify user exists
      command: getent passwd {{ username }}
      register: user_check
      changed_when: false

    - name: Show user record
      debug:
        msg: "{{ user_check.stdout }}"
EOF
```

Run:
```bash
ansible-playbook -i inventory/hosts playbooks/add-user.yml
```

### Idempotency Test
Run the same playbook twice. Second run shows `changed=0` on all nodes
— Ansible checked state, found everything correct, changed nothing.

```bash
ansible-playbook -i inventory/hosts playbooks/add-user.yml
# PLAY RECAP: changed=0 on all nodes
```

### Adding Different Users
Change the `vars` section — no other modifications needed:
```yaml
vars:
  username: hpcuser3
  uid: 1003
  gid: 1003
  fullname: "HPC Test User 3"
  group: hpcusers
```

Or pass variables at runtime:
```bash
ansible-playbook -i inventory/hosts playbooks/add-user.yml \
  -e "username=hpcuser3 uid=1003 gid=1003"
```

## 11.7 Playbook: Cluster Health Check

Check all critical services across the cluster in one command.

```bash
cat > /etc/ansible/hpc-lab/playbooks/cluster-health.yml << 'EOF'
---
- name: Check headnode services
  hosts: headnodes
  gather_facts: no

  tasks:
    - name: Check critical headnode services
      systemd:
        name: "{{ item }}"
      register: service_status
      loop:
        - slurmctld
        - slurmdbd
        - nfs-server
        - prometheus
        - grafana-server
        - munge
      changed_when: false

    - name: Show service status
      debug:
        msg: "{{ item.item }}: {{ item.status.ActiveState }}"
      loop: "{{ service_status.results }}"
      loop_control:
        label: "{{ item.item }}"

- name: Check compute node services
  hosts: compute
  gather_facts: no

  tasks:
    - name: Check critical compute services
      systemd:
        name: "{{ item }}"
      register: service_status
      loop:
        - slurmd
        - munge
        - node_exporter
      changed_when: false

    - name: Show service status
      debug:
        msg: "{{ item.item }}: {{ item.status.ActiveState }}"
      loop: "{{ service_status.results }}"
      loop_control:
        label: "{{ item.item }}"
EOF
```

Run:
```bash
ansible-playbook -i inventory/hosts playbooks/cluster-health.yml
```

Expected — all services show `active`:
```
slurmctld:    active
slurmdbd:     active
nfs-server:   active
prometheus:   active
grafana-server: active
munge:        active   (headnode)
slurmd:       active   (compute nodes)
munge:        active   (compute nodes)
node_exporter: active  (compute nodes)
```

## 11.8 Ad-Hoc Commands

For quick one-off tasks, use ad-hoc commands without a playbook:

```bash
# Run any command on all nodes
ansible cluster -m command -a "uptime"

# Check disk space on all nodes
ansible cluster -m command -a "df -h /"

# Check if a service is running
ansible cluster -m systemd -a "name=munge" --one-line

# Restart a service on compute nodes only
ansible compute -m systemd -a "name=slurmd state=restarted"

# Copy a file to all nodes
ansible cluster -m copy -a "src=/etc/hosts dest=/etc/hosts"

# Run a shell command (with pipes, redirects)
ansible cluster -m shell -a "ps aux | grep slurm | wc -l"
```

## 11.9 PLAY RECAP Reference

| Field       | Meaning                                      |
|-------------|----------------------------------------------|
| ok          | Tasks that ran and found correct state       |
| changed     | Tasks that actually modified something       |
| unreachable | Nodes that couldn't be contacted             |
| failed      | Tasks that encountered an error              |
| skipped     | Tasks skipped due to `when` conditions       |

Healthy run: `failed=0` and `unreachable=0` on all nodes.

## Scaling Pattern

As the cluster grows, expand playbooks to handle more nodes:
- Add nodes to inventory — playbooks automatically apply to them
- Variables in group_vars apply to all nodes in that group automatically
- Roles package reusable logic (nfs-client, slurmd, monitoring) that
  can be applied to any new node in one line

## Troubleshooting

### Connection refused / unreachable
```bash
# Test SSH manually
ssh root@compute-1 hostname
# Test Ansible ping
ansible compute-1 -m ping -vvv
```

### Task fails on one node only
Run with `-l` to limit to that node:
```bash
ansible-playbook playbooks/cluster-health.yml -l compute-1
```

### Changed=1 on every run (not idempotent)
The task is not using an idempotent module. Avoid `command` and `shell`
modules for state-changing tasks — use `user`, `group`, `copy`,
`systemd`, `dnf` modules instead. These check state before acting.

### Variables not resolving
Check variable syntax — Ansible uses `{{ variable_name }}` with spaces
inside the braces. Missing quotes around variables in YAML also causes
issues.
