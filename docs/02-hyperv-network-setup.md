# Phase 2: Hyper-V Network Setup

## Overview
Setting up virtual networking in Hyper-V before creating any VMs.

## Hyper-V Switch Types
| Type     | VM-to-VM | VM-to-Host | VM-to-External |
|----------|----------|------------|----------------|
| External | Yes      | Yes        | Yes            |
| Internal | Yes      | Yes        | No (unless NAT)|
| Private  | Yes      | No         | No             |

## Switches Created

### vSwitch1 (External) — Pre-existing
- Bound to physical NIC (Intel I210 secondary)
- Provides host internet connectivity
- NOT used by cluster VMs directly

### LabSwitch1 (Internal) — Pre-existing
- IP: 192.168.30.1/28 (on host)
- NAT configured via Windows NetNat
- Headnode connects here for internet access

### ClusterSwitch (Private) — Created for cluster
```powershell
New-VMSwitch -Name "ClusterSwitch" -SwitchType Private
