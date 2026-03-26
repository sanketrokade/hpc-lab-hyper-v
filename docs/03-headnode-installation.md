# Phase 3: Head Node VM Creation and OS Installation

## Overview
Creating the headnode VM in Hyper-V using PowerShell and installing Rocky Linux 9.7 Minimal.
All VM operations done via CLI for reproducibility and documentation.

## Why CLI Over GUI?
- **Reproducible** — run the same commands, get the same result
- **Documentable** — exact commands go straight into docs
- **Scriptable** — could automate all three VMs with one script
- **Auditable** — anyone reading the repo sees exactly what was done

## VM Creation

### Step 1: Create VM with OS disk
```powershell
New-VM -Name "headnode" -Generation 2 -MemoryStartupBytes 8GB `
  -Path "D:\sanket\VMs\cluster" `
  -NewVHDPath "D:\sanket\VMs\cluster\headnode\headnode-os.vhdx" `
  -NewVHDSizeBytes 20GB -SwitchName "LabSwitch1"
```

### Step 2: Set CPU count
```powershell
Set-VMProcessor -VMName "headnode" -Count 4
```

### Step 3: Disable dynamic memory
```powershell
Set-VMMemory -VMName "headnode" -DynamicMemoryEnabled $false
```
**Why static memory?** SLURM needs to know exactly how much RAM each node has.
Dynamic memory causes unpredictable resource availability and failed jobs.

### Step 4: Create and attach data disk
```powershell
New-VHD -Path "D:\sanket\VMs\cluster\headnode\headnode-data.vhdx" `
  -SizeBytes 50GB -Dynamic
Add-VMHardDiskDrive -VMName "headnode" `
  -Path "D:\sanket\VMs\cluster\headnode\headnode-data.vhdx"
```
**Why Dynamic for data disk in lab?** Saves physical SSD space.
In production HPC, use Fixed for predictable I/O latency — no runtime
allocation overhead, no fragmentation, contiguous blocks on disk.

### Step 5: Add second NIC for cluster network
```powershell
Add-VMNetworkAdapter -VMName "headnode" -SwitchName "ClusterSwitch" -Name "Cluster"
```

### Step 6: Set Secure Boot template for Linux
```powershell
Set-VMFirmware -VMName "headnode" `
  -SecureBootTemplate "MicrosoftUEFICertificateAuthority"
```
**Why change the template?** Default is `MicrosoftWindows` which only trusts
Windows bootloaders. `MicrosoftUEFICertificateAuthority` trusts third-party
signed bootloaders including Linux distributions.

### Step 7: Mount Rocky Linux ISO
```powershell
Add-VMDvdDrive -VMName "headnode" `
  -Path "D:\sanket\softwares\iso\Rocky-9.7-x86_64-minimal.iso"
```

### Step 8: Set boot order — DVD first
```powershell
$dvd = Get-VMDvdDrive -VMName "headnode"
Set-VMFirmware -VMName "headnode" -FirstBootDevice $dvd
```

### Step 9: Verification
```powershell
Get-VM -Name "headnode" | Format-List Name, Generation, MemoryStartup, ProcessorCount, State
Get-VMHardDiskDrive -VMName "headnode" | Format-Table Path, ControllerType -AutoSize
Get-VMNetworkAdapter -VMName "headnode" | Format-Table Name, SwitchName -AutoSize
Get-VMFirmware -VMName "headnode" | Format-List SecureBoot, SecureBootTemplate
```

**Verified output:**
- Generation: 2
- Memory: 8GB (8589934592 bytes)
- Processors: 4
- Two SCSI disks (20GB OS + 50GB data)
- Two NICs (LabSwitch1 + ClusterSwitch)
- SecureBoot: On, Template: MicrosoftUEFICertificateAuthority

## OS Installation

### Boot and connect
```powershell
Start-VM -Name "headnode"
vmconnect.exe localhost headnode
```

### Installer Settings

| Setting                  | Value                   |
|--------------------------|-------------------------|
| Installation Destination | 20GB disk only          |
| Storage Configuration    | Custom                  |
| Partitioning Scheme      | Standard Partition      |
| Root password            | Set                     |
| User creation            | Created (sanket, sudo)  |
| Network                  | Left unconfigured       |
| Time zone                | Set to local            |
| Software selection       | Minimal Install         |

**Important:** Do NOT touch the 50GB data disk during installation.
It is handled post-install with LVM.

### Partition Layout (20GB OS Disk — GPT/UEFI)

| Partition | Mount      | Size    | Filesystem           | Type     |
|-----------|------------|---------|----------------------|----------|
| sda1      | /boot/efi  | 600 MB  | EFI System Partition | Standard |
| sda2      | /boot      | 1 GB    | xfs                  | Standard |
| sda3      | swap       | 2 GB    | swap                 | Standard |
| sda4      | /          | ~16.4GB | xfs                  | Standard |

### Why 600MB for ESP?
Rocky default is 200MB. Kernel updates accumulate .efi files.
600MB gives headroom so the ESP doesn't fill up after a few months.

### Why Standard Partitions on OS Disk?
OS disk doesn't need the flexibility of LVM — it won't be resized.
LVM is reserved for the data disk where shared storage needs to grow.

### BIOS vs UEFI Partition Difference
- **BIOS/MBR:** Bootloader in first 512 bytes. No filesystem awareness.
- **UEFI/GPT:** Firmware understands FAT32, reads ESP, loads .efi files.
- **Why FAT32?** Universal compatibility — every OS vendor agreed on it.
- **GPT protective MBR:** Dummy MBR in first sector so legacy tools don't overwrite disk.

## Post-Install Verification
After reboot and login:
```bash
lsblk
# Confirms: sda (20GB, 4 partitions), sdb (50GB, no partitions)

cat /etc/fstab
# Confirms: UUID-based mounts for /, /boot, /boot/efi, swap

ip a
# Confirms: eth0 and eth1 present, no IPs assigned yet
```

## NIC Identification
Hyper-V synthetic NICs show as eth0/eth1 (not ens160/ens192 like VMware).
To identify which interface connects to which switch, cross-reference MAC addresses:

**From inside VM:**
```bash
ip a
# Shows MAC for each interface
```

**From host PowerShell:**
```powershell
Get-VMNetworkAdapter -VMName "headnode" | Format-Table Name, SwitchName, MacAddress -AutoSize
```

**Result:**
- eth0 (00:15:5d:59:08:03) → LabSwitch1
- eth1 (00:15:5d:59:08:04) → ClusterSwitch
