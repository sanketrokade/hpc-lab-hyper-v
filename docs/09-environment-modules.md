# Phase 9: Environment Modules (Lmod)

## Overview
Install and configure Lmod — the Lua-based module system used by most
major HPC centers. Lmod allows users to load, unload, and switch between
software versions without conflicts, without admin privileges, and without
affecting other users.

## The Problem Modules Solve

### Version Conflicts
A cluster serves many research groups simultaneously:
- Group A needs GCC 9.3 for legacy code
- Group B needs GCC 12.2 for new features
- Group C needs Python 3.8, Group D needs Python 3.11

Installing everything with `dnf` causes version conflicts. The system
can only have one `gcc` in `/usr/bin/gcc` at a time.

### What Modules Actually Do
`module load gcc/11.5.0` does exactly one thing — modifies environment
variables in the current shell session:

- **PATH** — prepends `/export/apps/gcc/11.5.0/bin` so `gcc` resolves correctly
- **LD_LIBRARY_PATH** — prepends library path for runtime linking
- **MANPATH** — adds man pages
- **Custom variables** — GCC_HOME, CC, CXX etc.

`module unload gcc/11.5.0` removes those paths cleanly.
`module swap gcc/9.3 gcc/12.2` removes one and adds the other atomically.

Nothing is installed. Nothing is compiled. Just PATH manipulation —
done cleanly, consistently, and reversibly.

### Modulefiles vs Software
| Component   | What it is           | Where it lives                    |
|-------------|----------------------|-----------------------------------|
| Software    | Actual binaries      | /export/apps/gcc/11.5.0/bin/      |
| Modulefile  | Text file with paths | /export/apps/modulefiles/gcc/     |

The modulefile is a recipe card. It tells Lmod where the ingredients
(binaries) are. The software itself is untouched.

## Why Lmod

- Written in Lua — fast, lightweight
- Handles module dependencies automatically
- Used by TACC, NERSC, and most major HPC centers
- Available in EPEL for Rocky Linux
- Supports both Lua (.lua) and TCL modulefiles

## Directory Structure

```
/export/apps/
├── modulefiles/
│   └── gcc/
│       └── 11.5.0.lua      ← modulefile
└── gcc/                     ← actual software (future)
    └── 11.5.0/
        ├── bin/
        └── lib/
```

Modulefiles live in `/export/apps/modulefiles/` alongside the software
they describe. Since `/export/apps/` is NFS shared and read-only on
compute nodes, modulefiles are automatically available everywhere.

## Prerequisites
- NFS `/export/apps/` mounted on all nodes (Phase 5 complete)
- EPEL repository available on all nodes
- All nodes reachable from headnode

## 9.1 Install Lmod

### On headnode

```bash
dnf install -y lua lua-libs lua-posix
dnf install -y Lmod
```

Note: Package name is `Lmod` (capital L) on Rocky Linux 9.

### On compute nodes

```bash
ssh compute-1 "dnf install -y Lmod"
ssh compute-2 "dnf install -y Lmod"
```

### Initialize Lmod in Current Shell

Lmod installs profile scripts in `/etc/profile.d/` that load automatically
for new login sessions. For the current session, source manually:

```bash
source /etc/profile.d/modules.sh
module --version
```

Expected output shows Lmod version and author information.

## 9.2 Configure Modulepath

Lmod defaults to `/etc/modulefiles` and `/usr/share/modulefiles`.
We need to add our custom path `/export/apps/modulefiles`.

### Create Modulefile Directory

```bash
mkdir -p /export/apps/modulefiles
```

### Add Custom Path System-Wide

```bash
cat > /etc/profile.d/01-cluster-modulepath.sh << 'EOF'
export MODULEPATH=/export/apps/modulefiles:$MODULEPATH
EOF
```

### Distribute to Compute Nodes

```bash
scp /etc/profile.d/01-cluster-modulepath.sh root@compute-1:/etc/profile.d/
scp /etc/profile.d/01-cluster-modulepath.sh root@compute-2:/etc/profile.d/
```

### Verify

```bash
source /etc/profile.d/01-cluster-modulepath.sh
module avail
```

`/export/apps/modulefiles` should appear in the output. Empty for now
since no modulefiles have been created yet.

## 9.3 Create a Modulefile

### Directory Structure for GCC

```bash
mkdir -p /export/apps/modulefiles/gcc
```

### Write the Modulefile

```bash
cat > /export/apps/modulefiles/gcc/11.5.0.lua << 'EOF'
-- GCC 11.5.0 modulefile
-- Loaded from system installation at /usr/bin

whatis("Name: GCC")
whatis("Version: 11.5.0")
whatis("Description: GNU Compiler Collection - C, C++ compilers")

help([[
GCC 11.5.0 - GNU Compiler Collection
Provides: gcc, g++, cpp, make

Usage:
  gcc hello.c -o hello
  g++ hello.cpp -o hello
]])

local gcc_root = "/usr"

prepend_path("PATH",            pathJoin(gcc_root, "bin"))
prepend_path("MANPATH",         pathJoin(gcc_root, "share/man"))
prepend_path("LD_LIBRARY_PATH", pathJoin(gcc_root, "lib64"))

setenv("GCC_HOME", gcc_root)
setenv("CC",  "gcc")
setenv("CXX", "g++")
EOF
```

### Modulefile Syntax Reference

| Function          | Purpose                                    |
|-------------------|--------------------------------------------|
| whatis()          | Short description shown in module avail    |
| help()            | Long help shown with module help           |
| prepend_path()    | Adds to beginning of environment variable  |
| append_path()     | Adds to end of environment variable        |
| setenv()          | Sets an environment variable               |
| unsetenv()        | Removes an environment variable            |
| load()            | Loads another module as dependency         |
| pathJoin()        | Joins path components safely               |

## 9.4 Test the Module

### Load and Verify

```bash
module avail
module load gcc/11.5.0
module list
echo $GCC_HOME
echo $CC
gcc --version
```

Expected:
```
Currently Loaded Modules:
  1) gcc/11.5.0

GCC_HOME = /usr
CC = gcc
gcc (GCC) 11.5.0 ...
```

### Unload and Verify Clean

```bash
module unload gcc/11.5.0
module list
echo $GCC_HOME
echo $CC
```

Expected: No modules loaded, GCC_HOME and CC empty.
Environment completely restored — no traces left.

### Get Module Information

```bash
module help gcc/11.5.0
module show gcc/11.5.0
module whatis gcc/11.5.0
```

## 9.5 Verify on Compute Nodes

Modulefiles live on NFS — no copying needed. Just verify Lmod
can see them from compute nodes:

```bash
ssh compute-1 "source /etc/profile.d/modules.sh && \
  source /etc/profile.d/01-cluster-modulepath.sh && \
  module avail"
```

`gcc/11.5.0` should appear in `/export/apps/modulefiles` section.

## 9.6 Test via SLURM Job

### As Root

```bash
sbatch --wrap="source /etc/profile.d/modules.sh && \
  source /etc/profile.d/01-cluster-modulepath.sh && \
  module load gcc/11.5.0 && gcc --version && echo \$GCC_HOME" \
  --job-name=moduletest -N 1

sleep 10
sacct --format=JobID,JobName,State,ExitCode,NodeList
```

### As Regular User

Regular users get module system automatically via `/etc/profile.d/`
on login — no manual sourcing needed:

```bash
su - hpcuser1
sbatch --wrap="module load gcc/11.5.0 && gcc --version && echo Compiler: \$CC" \
  --job-name=usermodule -N 1

sleep 10
sacct --format=JobID,JobName,State,ExitCode,NodeList,User
cat ~/slurm-*.out
```

Expected job output:
```
gcc (GCC) 11.5.0 ...
Compiler: gcc
```

## 9.7 Software Installation Pattern

In production, software installed into `/export/apps/` as standalone
builds — not relying on system packages:

```
# Install software
./configure --prefix=/export/apps/gcc/12.2.0
make && make install

# Create modulefile
cat > /export/apps/modulefiles/gcc/12.2.0.lua << 'EOF'
local gcc_root = "/export/apps/gcc/12.2.0"
prepend_path("PATH", pathJoin(gcc_root, "bin"))
prepend_path("LD_LIBRARY_PATH", pathJoin(gcc_root, "lib64"))
setenv("GCC_HOME", gcc_root)
setenv("CC", "gcc")
setenv("CXX", "g++")
EOF
```

Benefits:
- Binary lives in `/export/apps/` — NFS shared, no install on each node
- Multiple versions coexist cleanly
- Module controls which version each user sees
- `/export/apps/` exported read-only — users can't modify software

## Common Module Commands

| Command                    | Purpose                              |
|----------------------------|--------------------------------------|
| module avail               | List all available modules           |
| module list                | List currently loaded modules        |
| module load gcc/11.5.0     | Load a specific module               |
| module unload gcc/11.5.0   | Unload a module                      |
| module swap gcc/9.3 gcc/12 | Switch between versions              |
| module purge               | Unload all modules                   |
| module show gcc/11.5.0     | Show what a module does              |
| module help gcc/11.5.0     | Show module help text                |
| module spider gcc          | Search for modules by name           |
| ml                         | Short alias for module               |

## Troubleshooting

### module command not found
```bash
source /etc/profile.d/modules.sh
```

### Custom modulefiles not visible
Check MODULEPATH includes `/export/apps/modulefiles`:
```bash
echo $MODULEPATH
```
If missing, source the cluster modulepath file:
```bash
source /etc/profile.d/01-cluster-modulepath.sh
```

### Module loads but software not found
Software not installed on that compute node. Either:
1. Install software on all compute nodes
2. Compile and install into `/export/apps/` so it's NFS shared

### Duplicate paths in module avail output
Both `00-modulepath.sh` and `01-cluster-modulepath.sh` adding same path.
Cosmetic issue only — does not affect functionality.
