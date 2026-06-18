# FAILOVER-CLUSTER
# Hyper-V Failover Cluster Deployment Guide

## Environment Overview

| Component | Details |
|------------|------------|
| Platform | Windows Server 2022 Standard |
| Hypervisor | Hyper-V |
| Cluster Nodes | 3 |
| Domain | vss.local |
| Cluster Name | ICCC-HV-CLUSTER |
| Storage | NAS (iSCSI + MPIO) |
| CSV Volumes | 10TB + 40TB |
| Witness | 2GB Disk Witness |

---

# Architecture

```text
                         +----------------+
                         | DOMAIN SERVER  |
                         | 100.75.221.201 |
                         +-------+--------+
                                 |
                                 |
                +----------------+----------------+
                |                                 |
                |                                 |
      +---------+---------+             +---------+---------+
      | ICCC-HOST-1       |             | ICCC-HOST-2       |
      | 100.75.221.199    |             | 100.75.221.198    |
      +---------+---------+             +---------+---------+
                |                                 |
                +----------------+----------------+
                                 |
                                 |
                       +---------+---------+
                       | ICCC-HOST-3       |
                       | 100.75.221.197    |
                       +---------+---------+
                                 |
                                 |
                      =====================
                         Storage Network
                      =====================

         Controller A: 100.75.222.181
         Controller B: 100.75.222.182
```

---

# Network Design

## Management Team

| Team Name | MGMT_TEAMING |
|------------|------------|
| Mode | Switch Independent |
| Algorithm | Dynamic |
| Members | Ethernet, Ethernet 2 |
| Speed | 2 Gbps |

---

## Hyper-V Switch

```powershell
Get-VMSwitch
```

Output:

```text
MULTI_VLAN_SWITCH
```

---

# Storage Network

## Host Storage IPs

| Host | NIC1 | NIC2 |
|---------|---------|---------|
| HOST1 | 100.75.222.185 | 100.75.222.186 |
| HOST2 | 100.75.222.187 | 100.75.222.188 |
| HOST3 | 100.75.222.189 | 100.75.222.190 |

## NAS Controllers

| Controller | IP |
|------------|------------|
| Controller A | 100.75.222.181 |
| Controller B | 100.75.222.182 |

### Storage Rules

- No Gateway
- No DNS
- MPIO Enabled
- No NIC Teaming

---

# Windows Features

Install on all hosts:

```powershell
Install-WindowsFeature `
Hyper-V, `
Failover-Clustering, `
Multipath-IO, `
RSAT-Clustering, `
RSAT-Clustering-PowerShell, `
RSAT-Hyper-V-Tools `
-IncludeManagementTools
```

Reboot:

```powershell
Restart-Computer
```

---

# MPIO Configuration

Enable iSCSI support:

```powershell
Enable-WindowsOptionalFeature `
-Online `
-FeatureName MultiPathIO
```

GUI:

```text
MPIO
→ Discover Multi-Paths
→ Add support for iSCSI devices
→ Reboot
```

Verify:

```powershell
mpclaim -s -d
```

---

# iSCSI Configuration

Add portals:

```text
100.75.222.181
100.75.222.182
```

Enable:

```text
Enable Multi-path
```

Connect all required LUNs.

---

# Firewall Configuration

Enable required firewall groups:

```powershell
Enable-NetFirewallRule -DisplayGroup "Failover Clusters"
Enable-NetFirewallRule -DisplayGroup "Hyper-V"
Enable-NetFirewallRule -DisplayGroup "Windows Remote Management"
Enable-NetFirewallRule -DisplayGroup "Remote Service Management"
```

---

# Required Ports

| Port | Protocol | Purpose |
|---------|---------|---------|
| 3343 | TCP/UDP | Cluster Service |
| 6600 | TCP | Live Migration |
| 445 | TCP | SMB |
| 135 | TCP | RPC |
| 3260 | TCP | iSCSI |
| 5985 | TCP | WinRM |
| 5986 | TCP | WinRM HTTPS |

---

# Time Synchronization

## PDC Emulator

```powershell
w32tm /config /manualpeerlist:"pool.ntp.org" /syncfromflags:manual /reliable:yes /update
net stop w32time
net start w32time
w32tm /resync
```

## Cluster Hosts

```powershell
w32tm /config /syncfromflags:domhier /update
w32tm /resync
```

Verify:

```powershell
w32tm /query /source
```

Expected:

```text
DOMAIN-SERVER.vss.local
```

---

# Cluster Creation

Validation:

```powershell
Test-Cluster `
-Node ICCC-HOST-1,ICCC-HOST-2,ICCC-HOST-3
```

Create:

```powershell
New-Cluster `
-Name ICCC-HV-CLUSTER `
-Node ICCC-HOST-1,ICCC-HOST-2,ICCC-HOST-3 `
-StaticAddress 100.75.221.200
```

---

# Cluster Shared Volumes

| Disk | Size |
|---------|---------|
| Disk Witness | 2 GB |
| CSV Volume1 | 10 TB |
| CSV Volume2 | 40 TB |

Convert:

```powershell
Add-ClusterSharedVolume "Cluster Disk 2"
Add-ClusterSharedVolume "Cluster Disk 3"
```

---

# Live Migration

Authentication:

```text
CredSSP
```

Enable:

```text
Hyper-V Settings
→ Live Migrations
→ Enable incoming and outgoing live migrations
```

---

# VM Placement

## Volume1

```text
DOMAIN-SERVER
DOMAIN-SERVER-02
DB-SERVER
```

## Volume2

```text
WEB-API-SERVER
INT-SERVER
HM-SERVER
NOTIFY-SERVER
```

---

# Health Checks

```powershell
Get-ClusterNode
Get-ClusterGroup
Get-ClusterSharedVolume
Get-VM
Get-MPIOPath
```

---

# Troubleshooting

## Cluster Name Error 1311

Verify:

```powershell
nltest /dsgetdc:vss.local
Test-ComputerSecureChannel
```

---

## Storage Offline

```powershell
Get-Disk
Get-MPIOPath
```

---

## Live Migration Failure

Verify:

```powershell
Test-NetConnection HOST2 -Port 6600
Test-NetConnection HOST3 -Port 6600
```

---

# Production Status

- 3 Node Hyper-V Cluster
- Dual Controller NAS
- MPIO Enabled
- CSV Enabled
- Live Migration Enabled
- AD Integrated
- Primary + Secondary Domain Controllers
- Production Ready
