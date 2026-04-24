# Phase 12 — OpenMPI Installation and MPI Job Testing

## Overview
Install OpenMPI 4.1.8 from source, configure it for SSH-based launch, and verify
multi-node MPI job execution via SLURM across both compute nodes.

## Why OpenMPI from Source
- Production HPC clusters always compile MPI from source
- Allows control over which features, transports, and launchers are compiled in
- Installs to `/export/apps` — NFS-shared automatically to all compute nodes
- Version pinned via Lmod modulefile — multiple versions can coexist

## Why SSH-Based Launch (not srun)
SLURM 22.05 in Rocky Linux repos is built without PMI/PMIx support. This means
`srun` cannot bootstrap MPI processes. The correct approach is to use
`mpirun --mca plm rsh` which uses SSH to launch `orted` daemons on compute nodes
directly, bypassing SLURM's process launcher entirely while still respecting
SLURM's resource allocation.

## Prerequisites
Build dependencies installed on all three nodes:

```bash
dnf install -y gcc gcc-c++ gcc-gfortran make wget \
    perl perl-Data-Dumper \
    zlib-devel openssl-devel
```

## Installation (headnode only)

### Download and Extract
```bash
cd /tmp
wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.8.tar.gz
tar -xzf openmpi-4.1.8.tar.gz
cd openmpi-4.1.8
```

### Configure
```bash
./configure --prefix=/export/apps/openmpi/4.1.8 2>&1 | tail -5
```

No PMIx or PMI flags — SLURM on this cluster has no PMI support.

### Build and Install
```bash
make -j4 2>&1 | tail -3
make install 2>&1 | tail -3
```

### Verify
```bash
ls /export/apps/openmpi/4.1.8/bin/mpirun
```

## Lmod Modulefile

```bash
mkdir -p /export/apps/modulefiles/openmpi
cat > /export/apps/modulefiles/openmpi/4.1.8.lua << 'EOF'
help([[OpenMPI 4.1.8]])

whatis("Name: OpenMPI")
whatis("Version: 4.1.8")

local base = "/export/apps/openmpi/4.1.8"

prepend_path("PATH",            pathJoin(base, "bin"))
prepend_path("LD_LIBRARY_PATH", pathJoin(base, "lib"))
prepend_path("MANPATH",         pathJoin(base, "share/man"))
EOF
```

Verify:
```bash
module avail
module load openmpi/4.1.8
mpirun --version
```

## Firewall Fix on Compute Nodes

MPI daemons communicate over arbitrary TCP ports. Compute nodes only have the
cluster private network (eth0) which must be in the trusted zone to allow
unrestricted inter-node communication:

```bash
ssh compute-1 "firewall-cmd --zone=trusted --add-interface=eth0 --permanent && firewall-cmd --reload"
ssh compute-2 "firewall-cmd --zone=trusted --add-interface=eth0 --permanent && firewall-cmd --reload"
```

## Test Program

```c
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    char hostname[256];
    gethostname(hostname, 256);
    printf("Hello from rank %d of %d on %s\n", rank, size, hostname);
    MPI_Finalize();
    return 0;
}
```

Compile (as hpcuser1):
```bash
mpicc /tmp/hello_mpi.c -o /export/home/hpcuser1/hello_mpi
```

## SLURM Job Script

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
       /export/home/hpcuser1/hello_mpi
```

Submit and verify:
```bash
sbatch mpi_test.sh
cat slurm-<jobid>.out
```

## Expected Output
```
Hello from rank 0 of 4 on compute-1
Hello from rank 1 of 4 on compute-1
Hello from rank 2 of 4 on compute-2
Hello from rank 3 of 4 on compute-2
```

Note: `mca_btl_openib` warning about `librdmacm.so.1` is expected and harmless —
this lab has no InfiniBand hardware.

## Key Decisions

| Decision | Reason |
|----------|--------|
| OpenMPI 4.1.8 | Stable, widely used in production |
| Source install to /export/apps | NFS-shared automatically, version-controlled via Lmod |
| No PMIx/PMI compile flags | SLURM 22.05 from Rocky repos built without PMI support |
| --mca plm rsh | Forces SSH-based daemon launch, bypasses broken SLURM PMI |
| --prefix flag | Ensures compute nodes find orted without manual PATH changes |
| eth0 trusted zone on compute | Private cluster network needs no port restrictions |
| Jobs submitted as hpcuser1 | OpenMPI refuses to run as root by design |
