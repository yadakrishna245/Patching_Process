# 🖥️ VMware VM Patching Guide — Linux Guest OS in vCenter Environments

![Version](https://img.shields.io/badge/Version-2.0-blue)
![Platform](https://img.shields.io/badge/Platform-VMware%20vSphere%207.x%2F8.x-green)
![OS](https://img.shields.io/badge/OS-RHEL%206%2F7%2F8%2F9%2F10-red)
![Status](https://img.shields.io/badge/Status-Production--Ready-brightgreen)
![Last Updated](https://img.shields.io/badge/Updated-June%202026-orange)

> **Production-grade runbook for patching Linux virtual machines hosted on VMware vSphere/vCenter infrastructure.**

---

## 📋 Table of Contents

| # | Section | Description |
|---|---------|-------------|
| 1 | [Overview](#1-overview) | VM vs Physical patching comparison |
| 2 | [Pre-Requisites](#2-pre-requisites) | Access, tools, permissions |
| 3 | [Snapshot Management](#3-snapshot-management) | Create, manage, delete snapshots |
| 4 | [VMware Tools](#4-vmware-tools) | Validation and troubleshooting |
| 5 | [Pre-Patch Steps for VMs](#5-pre-patch-steps-for-vms) | VM-specific checks |
| 6 | [Patching Process](#6-patching-process) | yum/dnf commands per RHEL version |
| 7 | [DRS and HA Awareness](#7-drs-and-ha-awareness) | Cluster-aware patching |
| 8 | [Rollback via Snapshot](#8-rollback-via-snapshot) | Revert procedures |
| 9 | [Post-Patch Cleanup](#9-post-patch-cleanup) | Snapshot removal, validation |
| 10 | [ESXi Host Patching](#10-esxi-host-patching) | Lifecycle Manager, baselines |
| 11 | [vCenter Appliance Patching](#11-vcenter-appliance-patching) | VAMI update process |
| 12 | [Automation](#12-automation) | Ansible & PowerCLI scripts |
| 13 | [Troubleshooting](#13-troubleshooting) | Common issues & fixes |
| 14 | [Best Practices & Common Mistakes](#14-best-practices--common-mistakes) | Do's and Don'ts |
| 15 | [Quick Reference Commands](#15-quick-reference-commands) | Cheat sheet |

---

## 1. Overview

### What's the Same (VM vs Physical)?

| Aspect | Same? | Notes |
|--------|-------|-------|
| Package manager commands | ✅ | `yum`/`dnf` identical inside the guest |
| Kernel updates | ✅ | Same process, same reboot requirement |
| Repository configuration | ✅ | Satellite/RHSM works the same |
| Service validation | ✅ | `systemctl` checks are identical |
| Security compliance scans | ✅ | OpenSCAP/CIS benchmarks unchanged |

### What's Different (VM-Specific)?

| Aspect | Difference |
|--------|-----------|
| **Rollback** | Snapshot revert available (not possible on physical) |
| **Pre-patch safety net** | Take VM snapshot before patching |
| **VMware Tools** | Must be validated/updated alongside OS patches |
| **Hardware version** | VM hardware compatibility affects features |
| **Storage** | Thin-provisioned disks grow; snapshots consume datastore space |
| **Networking** | Virtual NIC driver (vmxnet3) may need attention after kernel updates |
| **Cluster awareness** | DRS/HA/vMotion considerations during maintenance |
| **ESXi dependency** | Host patches may be needed before guest OS patches |

> 💡 **Key Insight**: The guest OS patching process is identical. The *wrapper* around it (snapshot, validate tools, cleanup) is what makes VM patching different.

---

## 2. Pre-Requisites

### 2.1 Access Requirements

| Requirement | Details |
|-------------|---------|
| vCenter access | Web Client or API access to vCenter Server |
| Permissions | `VirtualMachine.State.CreateSnapshot`, `VirtualMachine.State.RemoveSnapshot`, `VirtualMachine.State.RevertToSnapshot` |
| SSH to guest | Root or sudo access to the Linux VM |
| Datastore access | Read access to verify free space |

### 2.2 Required Roles (Minimum)

```
Role: VM Patching Operator
Privileges:
  - Virtual Machine > Snapshot Management > Create/Delete/Revert
  - Virtual Machine > Interaction > Power Off/On/Reset
  - Virtual Machine > Guest Operations > Guest Operation Queries
  - Datastore > Browse Datastore
  - Host > Maintenance Mode (for ESXi patching only)
```

### 2.3 PowerCLI Setup

```powershell
# Install VMware PowerCLI
Install-Module -Name VMware.PowerCLI -Scope CurrentUser -Force

# Skip certificate warnings (lab/non-prod)
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false

# Connect to vCenter
Connect-VIServer -Server vcenter.example.com -User admin@vsphere.local -Password 'P@ssw0rd'

# Verify connection
Get-VIServer
```

**Expected Output:**
```
Name                           Port  User
----                           ----  ----
vcenter.example.com            443   VSPHERE.LOCAL\admin
```

### 2.4 govc Setup

```bash
# Install govc
curl -L -o govc.gz https://github.com/vmware/govmomi/releases/latest/download/govc_Linux_x86_64.tar.gz
tar -xzf govc.gz govc
sudo mv govc /usr/local/bin/
chmod +x /usr/local/bin/govc

# Set environment variables
export GOVC_URL="https://vcenter.example.com/sdk"
export GOVC_USERNAME="admin@vsphere.local"
export GOVC_PASSWORD="P@ssw0rd"
export GOVC_INSECURE=true  # Only for self-signed certs

# Verify connectivity
govc about
```

**Expected Output:**
```
FullName:     VMware vCenter Server 8.0.2 build-22617221
Name:         VMware vCenter Server
Version:      8.0.2
```

> ⚠️ **Security Note**: Never hardcode passwords. Use credential stores, environment variables from vault, or `govc session.login` with SSO tokens.

---


## 3. Snapshot Management

### 3.1 Why Snapshot Before Patching?

Snapshots provide an **instant rollback point** if patching causes:
- Kernel panic / boot failure
- Application incompatibility
- Network driver issues
- Service failures

### 3.2 Create Pre-Patch Snapshot

#### Using govc:
```bash
# Create snapshot with memory (powered-on VM)
govc snapshot.create -vm "/Datacenter/vm/Linux/prod-web-01" -m=true "Pre-Patch-$(date +%Y%m%d)"

# Create snapshot without memory (quiesced, requires VMware Tools)
govc snapshot.create -vm "prod-web-01" -m=false -q=true "Pre-Patch-$(date +%Y%m%d)"

# Verify snapshot was created
govc snapshot.tree -vm "prod-web-01"
```

**Expected Output:**
```
.
└── Pre-Patch-20260623
```

#### Using PowerCLI:
```powershell
# Single VM snapshot
$vm = Get-VM -Name "prod-web-01"
New-Snapshot -VM $vm -Name "Pre-Patch-$(Get-Date -Format yyyyMMdd)" -Description "Before June 2026 patching" -Memory -Quiesce

# Bulk snapshot for multiple VMs
$vmList = Get-Content "C:\patch-list.txt"
foreach ($vmName in $vmList) {
    $vm = Get-VM -Name $vmName
    New-Snapshot -VM $vm -Name "Pre-Patch-$(Get-Date -Format yyyyMMdd)" -Memory -Quiesce
    Write-Host "Snapshot created for $vmName" -ForegroundColor Green
}
```

#### Using vCenter GUI:
1. Right-click VM → **Snapshots** → **Take Snapshot**
2. Name: `Pre-Patch-YYYYMMDD`
3. Description: `Before scheduled patching window - ticket CHG00012345`
4. ☑️ Snapshot the virtual machine's memory
5. ☑️ Quiesce guest file system (requires VMware Tools)
6. Click **OK**

### 3.3 Snapshot Best Practices

| Practice | Recommendation |
|----------|---------------|
| Naming convention | `Pre-Patch-YYYYMMDD-TICKET#` |
| Include memory? | Yes for critical VMs; No for bulk patching (faster) |
| Quiesce? | Yes (ensures filesystem consistency) |
| Max snapshot age | **Delete within 24-72 hours** |
| Max snapshot depth | Never exceed 3 levels deep |
| Datastore free space | Ensure ≥ 20% free before snapshot |

### 3.4 Risks of Keeping Snapshots Too Long

| Risk | Impact |
|------|--------|
| **Performance degradation** | Each I/O must traverse snapshot delta files |
| **Datastore exhaustion** | Delta VMDKs grow continuously with writes |
| **Consolidation failure** | Large deltas may fail to merge, requiring downtime |
| **Backup failures** | Backup solutions may fail on VMs with stale snapshots |
| **Corruption risk** | Extended snapshot chains increase corruption probability |

> 🚨 **Critical Rule**: Snapshots are NOT backups. Delete within 72 hours maximum. Set calendar reminders.

### 3.5 List and Monitor Existing Snapshots

```bash
# govc - find VMs with snapshots
govc snapshot.tree -vm "prod-web-01"

# govc - find ALL VMs with snapshots in a folder
for vm in $(govc find /Datacenter/vm/Linux -type m); do
    snaps=$(govc snapshot.tree -vm "$vm" 2>/dev/null)
    if [ -n "$snaps" ]; then
        echo "=== $vm ==="
        echo "$snaps"
    fi
done
```

```powershell
# PowerCLI - Find all VMs with snapshots older than 3 days
Get-VM | Get-Snapshot | Where-Object { $_.Created -lt (Get-Date).AddDays(-3) } |
    Select-Object VM, Name, Created, SizeGB |
    Sort-Object Created |
    Format-Table -AutoSize
```

---

## 4. VMware Tools

### 4.1 Why VMware Tools Matter for Patching

- **Quiesced snapshots** require VMware Tools running
- **Guest OS information** (IP, hostname) reported to vCenter
- **Graceful shutdown** commands depend on Tools
- **vmxnet3 driver** performance depends on Tools version

### 4.2 Check VMware Tools Status

#### From inside the VM:
```bash
# Check if open-vm-tools is installed
rpm -qa | grep open-vm-tools

# Expected output:
# open-vm-tools-12.3.5-1.el8.x86_64

# Check service status
systemctl status vmtoolsd

# Check tools version
vmware-toolbox-cmd -v

# Expected output:
# 12.3.5.37


```

#### From vCenter (govc):
```bash
govc vm.info -json "prod-web-01" | jq '.virtualMachines[0].guest.toolsStatus'
# Expected: "toolsOk"

govc vm.info "prod-web-01" | grep -i "tools"
# Expected: Guest Tools: running (version 12355)
```

#### From PowerCLI:
```powershell
Get-VM "prod-web-01" | Select-Object Name,
    @{N="ToolsStatus";E={$_.ExtensionData.Guest.ToolsStatus}},
    @{N="ToolsVersion";E={$_.ExtensionData.Guest.ToolsVersionStatus2}}
```

### 4.3 Update open-vm-tools

```bash
# RHEL 7/8/9/10
sudo yum update open-vm-tools -y    # RHEL 7
sudo dnf update open-vm-tools -y    # RHEL 8/9/10

# Restart the service
sudo systemctl restart vmtoolsd

# Verify
vmware-toolbox-cmd -v
```

### 4.4 Troubleshooting VMware Tools

| Symptom | Cause | Fix |
|---------|-------|-----|
| Tools status: "not running" | vmtoolsd service stopped | `systemctl start vmtoolsd && systemctl enable vmtoolsd` |
| Tools status: "not installed" | Package missing | `yum install open-vm-tools -y` |
| Quiesce fails | Tools outdated or crashed | Update tools, restart vmtoolsd |
| No IP shown in vCenter | Tools not reporting | Restart vmtoolsd, check `/etc/vmware-tools/tools.conf` |
| Tools version mismatch | OS tools newer than bundled | Use open-vm-tools (recommended over bundled) |

> 💡 **Best Practice**: Always use `open-vm-tools` from the OS repo rather than VMware-bundled tools. It's the VMware-recommended approach since vSphere 7.0.

---


## 5. Pre-Patch Steps for VMs

### 5.1 Datastore Free Space Check

```bash
# govc - Check datastore usage
govc datastore.info "Datastore-SAN-01"
```

**Expected Output:**
```
Name:        Datastore-SAN-01
  Path:      /Datacenter/datastore/Datastore-SAN-01
  Type:      VMFS
  URL:       ds:///vmfs/volumes/60a5b3c2-1234abcd/
  Capacity:  2.0 TB
  Free:      680.5 GB (33%)
```

```powershell
# PowerCLI - Check all datastores below 20% free
Get-Datastore | Where-Object { ($_.FreeSpaceGB / $_.CapacityGB) -lt 0.20 } |
    Select-Object Name, @{N="CapacityGB";E={[math]::Round($_.CapacityGB,1)}},
    @{N="FreeGB";E={[math]::Round($_.FreeSpaceGB,1)}},
    @{N="FreePercent";E={[math]::Round(($_.FreeSpaceGB/$_.CapacityGB)*100,1)}} |
    Format-Table -AutoSize
```

> ⚠️ **Rule**: Do NOT proceed with snapshots if datastore free space is below 20%. Snapshots grow with every write operation.

### 5.2 Consolidate Old Snapshots

```bash
# Check for VMs needing consolidation
govc vm.info "prod-web-01" | grep -i "consolidation"

# If consolidation needed:
govc snapshot.remove -vm "prod-web-01" -consolidate
```

```powershell
# Find VMs needing disk consolidation
Get-VM | Where-Object { $_.ExtensionData.Runtime.ConsolidationNeeded } |
    Select-Object Name, PowerState
```

### 5.3 Verify VM Hardware Version

```bash
govc vm.info "prod-web-01" | grep "HW Version"
# Expected: HW Version: vmx-19 (ESXi 7.0 U2+)
```

| Hardware Version | ESXi Compatibility | Notes |
|-----------------|-------------------|-------|
| vmx-21 | ESXi 8.0 U2+ | Latest features |
| vmx-20 | ESXi 8.0+ | vTPM, VBS |
| vmx-19 | ESXi 7.0 U2+ | Recommended for most |
| vmx-14 | ESXi 6.7+ | Legacy |
| vmx-11 | ESXi 6.0+ | End of life |

### 5.4 Check DRS/HA Cluster Status

```powershell
# Check cluster health
Get-Cluster | Select-Object Name, DrsEnabled, DrsMode, HAEnabled, HAAdmissionControlEnabled

# Check for any HA errors
Get-Cluster "Production-Cluster" | Get-View | 
    Select-Object -ExpandProperty DrsFault
```

```bash
# govc - cluster info
govc cluster.info "/Datacenter/host/Production-Cluster"
```

### 5.5 Pre-Patch Checklist (VM-Specific)

- [ ] Datastore free space ≥ 20%
- [ ] No existing stale snapshots on target VMs
- [ ] VMware Tools running and current
- [ ] No active vMotion events on target VMs
- [ ] No ongoing storage migrations
- [ ] VM hardware version is supported
- [ ] Change ticket approved and window confirmed
- [ ] Backup completed (Veeam/CommVault job verified)

---

## 6. Patching Process

> 📌 **Note**: The guest OS patching commands are identical to physical servers. The difference is the snapshot safety net wrapping the process.

### 6.1 Patching Workflow for VMs

```
[Create Snapshot] → [Patch Guest OS] → [Validate] → [Delete Snapshot]
        ↑                                    ↓
        └────── [Revert Snapshot] ←── [FAILURE]
```

### 6.2 RHEL 6 (End of Life — Emergency Only)

```bash
# Check current version
cat /etc/redhat-release
# Red Hat Enterprise Linux Server release 6.10 (Santiago)

# Update all packages
sudo yum update -y

# Security updates only
sudo yum update --security -y

# Check if reboot needed
needs-restarting -r || echo "REBOOT REQUIRED"
```

### 6.3 RHEL 7 (Extended Life Phase)

```bash
# Check current version
cat /etc/redhat-release

# List available updates
sudo yum check-update

# Apply all updates
sudo yum update -y

# Security only
sudo yum update --security -y

# Kernel-only update
sudo yum update kernel -y

# Verify
rpm -qa --last | head -20
needs-restarting -r
```

### 6.4 RHEL 8

```bash
# Check current version
cat /etc/redhat-release

# List available updates
sudo dnf check-update

# Apply all updates
sudo dnf update -y

# Security advisories only
sudo dnf update --security -y

# Specific advisory
sudo dnf update --advisory=RHSA-2026:1234 -y

# Verify
dnf history info last
needs-restarting -r
```

### 6.5 RHEL 9

```bash
# Check current version
cat /etc/redhat-release
# Red Hat Enterprise Linux release 9.4 (Plow)

# Apply all updates
sudo dnf update -y

# Security only
sudo dnf update --security -y

# Exclude specific packages
sudo dnf update -y --exclude=kernel* --exclude=java*

# Verify installed updates
sudo dnf history info last
sudo needs-restarting -r
```

### 6.6 RHEL 10

```bash
# Check current version
cat /etc/redhat-release
# Red Hat Enterprise Linux release 10.0 (Coughlan)

# Apply all updates
sudo dnf update -y

# Security updates only
sudo dnf update --security -y

# Check reboot requirement
sudo dnf needs-restarting -r
# Exit code 1 = reboot needed, 0 = no reboot needed

# Reboot if required
sudo systemctl reboot
```

### 6.7 Post-Reboot Validation (All Versions)

```bash
# Verify new kernel loaded
uname -r

# Check uptime (should be minutes)
uptime

# Verify services are running
systemctl is-system-running
# Expected: running

# Check for failed services
systemctl --failed

# Verify network
ip addr show
ping -c 3 gateway_ip

# Verify application services
systemctl status httpd   # or your app service
systemctl status sshd
```

---


## 7. DRS and HA Awareness

### 7.1 Understanding the Impact

| Concept | Relevance to Patching |
|---------|----------------------|
| **DRS** (Distributed Resource Scheduler) | May vMotion VMs during patching — disable temporarily if needed |
| **HA** (High Availability) | Monitors heartbeats; reboot during patch may trigger HA event |
| **vMotion** | Live migration — avoid during snapshot operations |
| **Maintenance Mode** | Required for ESXi host patching, not guest OS patching |

### 7.2 Patching VMs in HA Clusters

```
Strategy: Patch one cluster node's VMs at a time

Cluster: 4 ESXi hosts, 40 VMs total
├── Host 1: Patch 10 VMs → Validate → Next
├── Host 2: Patch 10 VMs → Validate → Next
├── Host 3: Patch 10 VMs → Validate → Next
└── Host 4: Patch 10 VMs → Validate → Done
```

### 7.3 Disable DRS During Patching (Optional)

```powershell
# Set DRS to manual (prevents automatic vMotion)
$cluster = Get-Cluster "Production-Cluster"
Set-Cluster -Cluster $cluster -DrsAutomationLevel Manual -Confirm:$false

# After patching, re-enable
Set-Cluster -Cluster $cluster -DrsAutomationLevel FullyAutomated -Confirm:$false
```

> 💡 **Tip**: You don't always need to disable DRS. Only disable if you're patching a large batch and want predictable VM placement during the window.

### 7.4 vMotion Considerations

- **Do NOT** take snapshots during an active vMotion
- **Do NOT** vMotion a VM that has an active snapshot consolidation
- Verify no Storage vMotion is in progress before snapshot operations

```powershell
# Check for active vMotion tasks
Get-Task | Where-Object { $_.Name -like "*Relocate*" -or $_.Name -like "*Migrate*" } |
    Where-Object { $_.State -eq "Running" }
```

### 7.5 Maintenance Mode for ESXi Host Patching

```bash
# Put ESXi host in maintenance mode (evacuates VMs via vMotion)
govc host.maintenance.enter -host "esxi-host-01.example.com"

# Verify host is in maintenance mode
govc host.info "esxi-host-01.example.com" | grep "Maintenance"

# Exit maintenance mode after host patching
govc host.maintenance.exit -host "esxi-host-01.example.com"
```

> ⚠️ **Important**: Maintenance mode is for ESXi HOST patching only. Guest OS patching does NOT require the host to be in maintenance mode.

### 7.6 Patching Strategy for HA Clusters

| Scenario | Strategy |
|----------|----------|
| 2-node cluster | Patch one node's VMs, validate, then the other |
| N+1 cluster | Can patch multiple hosts' VMs simultaneously |
| Stretched cluster | Patch one site at a time |
| Critical app cluster | Patch standby/passive node first, failover, then patch former active |

---

## 8. Rollback via Snapshot

### 8.1 When to Revert

| Condition | Action |
|-----------|--------|
| VM won't boot after patch | ✅ Revert immediately |
| Critical application failure | ✅ Revert, investigate later |
| Network completely lost | ✅ Revert (via vCenter console) |
| Minor service issue | ❌ Fix forward — don't revert for small issues |
| Kernel panic on boot | ✅ Revert immediately |
| Performance degradation | ⚠️ Investigate first, revert if unresolvable in window |

### 8.2 Step-by-Step Revert

#### Using govc:
```bash
# List available snapshots
govc snapshot.tree -vm "prod-web-01"

# Revert to specific snapshot
govc snapshot.revert -vm "prod-web-01" "Pre-Patch-20260623"

# Power on if VM was powered off during revert
govc vm.power -on "prod-web-01"
```

#### Using PowerCLI:
```powershell
# Get the snapshot
$vm = Get-VM "prod-web-01"
$snapshot = Get-Snapshot -VM $vm -Name "Pre-Patch-20260623"

# Revert
Set-VM -VM $vm -Snapshot $snapshot -Confirm:$false

# Verify power state
Get-VM "prod-web-01" | Select-Object Name, PowerState
```

#### Using vCenter GUI:
1. Right-click VM → **Snapshots** → **Manage Snapshots**
2. Select `Pre-Patch-20260623`
3. Click **Revert to**
4. Confirm the operation
5. Verify VM power state and connectivity

### 8.3 Validation After Revert

```bash
# Verify kernel is back to original
uname -r
# Should show pre-patch kernel

# Verify patching was undone
rpm -qa --last | head -5
# Should NOT show today's patches

# Verify services
systemctl is-system-running

# Verify network
ip addr show
ping -c 3 gateway_ip

# Verify application
curl -s http://localhost:8080/health
```

### 8.4 Post-Revert Actions

1. **Document the failure** — What went wrong and why
2. **Delete the snapshot** — Even after revert, delete the snapshot (it's been consumed)
3. **Reschedule patching** — With fixes for the identified issue
4. **Update the change ticket** — Note rollback performed

---


## 9. Post-Patch Cleanup

### 9.1 Delete Snapshots (Critical — Do Within 24-72 Hours)

#### Using govc:
```bash
# Delete specific snapshot
govc snapshot.remove -vm "prod-web-01" "Pre-Patch-20260623"

# Delete ALL snapshots for a VM (use with caution)
govc snapshot.remove -vm "prod-web-01" "*"
```

#### Using PowerCLI:
```powershell
# Remove specific snapshot
Get-VM "prod-web-01" | Get-Snapshot -Name "Pre-Patch-20260623" | Remove-Snapshot -Confirm:$false

# Bulk remove snapshots older than 3 days
Get-VM | Get-Snapshot | Where-Object { $_.Created -lt (Get-Date).AddDays(-3) } | ForEach-Object {
    Write-Host "Removing snapshot '$($_.Name)' from VM '$($_.VM.Name)' (Age: $([math]::Round(((Get-Date) - $_.Created).TotalDays,1)) days)"
    Remove-Snapshot -Snapshot $_ -Confirm:$false
}
```

### 9.2 Verify VMware Tools Post-Patch

```bash
# Verify tools are still running after OS update
systemctl status vmtoolsd

# If tools were updated, verify version
vmware-toolbox-cmd -v

# Check vCenter reports correct status
govc vm.info "prod-web-01" | grep -i tool
```

### 9.3 Validate Network and Storage

```bash
# Network validation
ip addr show
ip route show
cat /etc/resolv.conf
ping -c 3 $(ip route | grep default | awk '{print $3}')

# Storage validation
df -h
mount | grep -v tmpfs
lsblk

# Verify NFS/CIFS mounts if applicable
showmount -e nfs-server.example.com
mount -a
```

### 9.4 Post-Patch Cleanup Checklist

- [ ] Snapshot deleted (within 72 hours max)
- [ ] VMware Tools running and reporting to vCenter
- [ ] Network connectivity verified (internal + external)
- [ ] Storage mounts intact
- [ ] Application services running
- [ ] Monitoring agents reporting (Zabbix/Nagios/Datadog)
- [ ] Change ticket updated with completion status
- [ ] DRS automation level restored (if changed)

---

## 10. ESXi Host Patching

> ⚠️ **Important**: ESXi host patching is **separate** from guest OS patching. This section covers the hypervisor layer.

### 10.1 vCenter Lifecycle Manager (vLCM)

vLCM (formerly Update Manager/VUM) manages ESXi host patching centrally.

#### Setup Baselines:
1. Navigate to **Menu** → **Lifecycle Manager**
2. Go to **Baselines** tab
3. Create new baseline: **Critical Host Patches (June 2026)**
4. Select patches or use dynamic criteria

### 10.2 Remediation Process

```
[Create Baseline] → [Attach to Cluster] → [Scan] → [Stage] → [Remediate]
```

#### PowerCLI Remediation:
```powershell
# Scan cluster for compliance
$cluster = Get-Cluster "Production-Cluster"
$baseline = Get-Baseline -Name "Critical Host Patches Q2-2026"

# Attach baseline
Attach-Baseline -Entity $cluster -Baseline $baseline

# Scan for compliance
Scan-Inventory -Entity $cluster

# Check compliance
Get-Compliance -Entity (Get-VMHost -Location $cluster) -Baseline $baseline |
    Select-Object Entity, Status
```

### 10.3 Rolling Update Strategy

```
Cluster: 4 ESXi Hosts
├── Host 1: Maintenance Mode → Patch → Reboot → Exit Maintenance ✓
├── Host 2: Maintenance Mode → Patch → Reboot → Exit Maintenance ✓
├── Host 3: Maintenance Mode → Patch → Reboot → Exit Maintenance ✓
└── Host 4: Maintenance Mode → Patch → Reboot → Exit Maintenance ✓

DRS automatically evacuates VMs before each host enters maintenance mode.
```

### 10.4 ESXi Patch via CLI (esxcli)

```bash
# SSH to ESXi host (if needed)
ssh root@esxi-host-01.example.com

# List available patches in depot
esxcli software sources profile list -d /vmfs/volumes/datastore1/patches/VMware-ESXi-8.0U2-depot.zip

# Install patch
esxcli software profile update -d /vmfs/volumes/datastore1/patches/VMware-ESXi-8.0U2-depot.zip -p ESXi-8.0U2-22617221-standard

# Verify
esxcli system version get

# Reboot
esxcli system shutdown reboot -r "ESXi patching June 2026"
```

---

## 11. vCenter Appliance Patching

### 11.1 VAMI Interface

Access: `https://vcenter.example.com:5480`

Login with: `root` credentials (not SSO admin)

### 11.2 Update Process

#### Via VAMI GUI:
1. Login to VAMI (`https://<vcenter-fqdn>:5480`)
2. Navigate to **Update** section
3. Click **Check Updates** → **Check Repository** or **Check URL**
4. Review available updates
5. Click **Stage and Install**
6. Monitor progress (can take 30-60 minutes)
7. Appliance will reboot automatically

#### Via CLI (appliance shell):
```bash
# SSH to vCenter appliance
ssh root@vcenter.example.com

# Check for updates
software-packages stage --url --acceptEulas

# List staged updates
software-packages list --staged

# Install staged updates
software-packages install --staged

# Check current version
cat /etc/applmgmt/appliance/update.conf
```

### 11.3 Pre-Update Checklist for vCenter

- [ ] Take file-based backup of VCSA (or snapshot at VM level)
- [ ] Verify current VCSA health: `https://<vcenter>:5480` → Health
- [ ] Ensure no active tasks in vCenter (migrations, provisioning)
- [ ] Verify external PSC connectivity (if applicable)
- [ ] Notify team — vCenter will be unavailable during update
- [ ] Schedule during lowest-activity window

> 🚨 **Warning**: vCenter outage affects management plane only. Running VMs are NOT affected. But you lose ability to manage VMs during the update.

---


## 12. Automation

### 12.1 Ansible Playbook — Snapshot + Patch + Cleanup

```yaml
---
# playbook: vmware_patch_workflow.yml
# Description: Complete VM patching workflow with snapshot safety net
# Usage: ansible-playbook vmware_patch_workflow.yml -i inventory/production -l patch_group_1

- name: "Phase 1: Create Pre-Patch Snapshots"
  hosts: localhost
  gather_facts: false
  vars:
    vcenter_hostname: "vcenter.example.com"
    vcenter_username: "admin@vsphere.local"
    vcenter_password: "{{ vault_vcenter_password }}"
    snapshot_name: "Pre-Patch-{{ ansible_date_time.date }}"
    vm_list: "{{ groups['patch_group_1'] }}"
  tasks:
    - name: Create snapshot for each VM
      community.vmware.vmware_guest_snapshot:
        hostname: "{{ vcenter_hostname }}"
        username: "{{ vcenter_username }}"
        password: "{{ vcenter_password }}"
        validate_certs: false
        datacenter: "Datacenter"
        name: "{{ item }}"
        state: present
        snapshot_name: "{{ snapshot_name }}"
        description: "Pre-patch snapshot - CHG00012345"
        memory_dump: true
        quiesce: true
      loop: "{{ vm_list }}"
      register: snapshot_results

    - name: Verify all snapshots created
      ansible.builtin.debug:
        msg: "Snapshot created for {{ item.item }}: {{ item.changed }}"
      loop: "{{ snapshot_results.results }}"
      when: item.changed

- name: "Phase 2: Patch Guest OS"
  hosts: patch_group_1
  become: true
  gather_facts: true
  serial: 5  # Patch 5 VMs at a time
  tasks:
    - name: Update all packages (RHEL 8/9/10)
      ansible.builtin.dnf:
        name: "*"
        state: latest
        security: true
      when: ansible_distribution_major_version | int >= 8
      register: patch_result

    - name: Update all packages (RHEL 7)
      ansible.builtin.yum:
        name: "*"
        state: latest
        security: true
      when: ansible_distribution_major_version | int == 7
      register: patch_result_7

    - name: Check if reboot is required
      ansible.builtin.command: needs-restarting -r
      register: reboot_check
      failed_when: false
      changed_when: reboot_check.rc == 1

    - name: Reboot if required
      ansible.builtin.reboot:
        reboot_timeout: 600
        msg: "Rebooting for kernel/system updates"
      when: reboot_check.rc == 1

    - name: Validate post-patch
      ansible.builtin.command: systemctl is-system-running
      register: system_status
      retries: 3
      delay: 30
      until: system_status.stdout == "running"

- name: "Phase 3: Cleanup Snapshots"
  hosts: localhost
  gather_facts: false
  vars:
    vcenter_hostname: "vcenter.example.com"
    vcenter_username: "admin@vsphere.local"
    vcenter_password: "{{ vault_vcenter_password }}"
    snapshot_name: "Pre-Patch-{{ ansible_date_time.date }}"
    vm_list: "{{ groups['patch_group_1'] }}"
  tasks:
    - name: Wait for validation period (optional)
      ansible.builtin.pause:
        prompt: "Patching complete. Press ENTER to delete snapshots or Ctrl+C to keep them for manual review."

    - name: Remove snapshots
      community.vmware.vmware_guest_snapshot:
        hostname: "{{ vcenter_hostname }}"
        username: "{{ vcenter_username }}"
        password: "{{ vcenter_password }}"
        validate_certs: false
        datacenter: "Datacenter"
        name: "{{ item }}"
        state: absent
        snapshot_name: "{{ snapshot_name }}"
      loop: "{{ vm_list }}"
```

### 12.2 PowerCLI Script — Bulk Snapshot Operations

```powershell
<#
.SYNOPSIS
    Bulk VM Snapshot Management for Patching
.DESCRIPTION
    Creates, monitors, and removes snapshots for a list of VMs during patching windows.
.USAGE
    .\Invoke-PatchSnapshots.ps1 -Action Create -VMListFile "C:\patch-vms.txt" -TicketNumber "CHG00012345"
    .\Invoke-PatchSnapshots.ps1 -Action Remove -VMListFile "C:\patch-vms.txt" -MaxAgeDays 3
    .\Invoke-PatchSnapshots.ps1 -Action Report
#>
param(
    [ValidateSet("Create","Remove","Report","Revert")]
    [string]$Action = "Report",
    [string]$VMListFile,
    [string]$TicketNumber = "CHG-UNKNOWN",
    [int]$MaxAgeDays = 3
)

# Connect to vCenter
$vcServer = "vcenter.example.com"
if (-not $global:DefaultVIServer) {
    Connect-VIServer -Server $vcServer
}

switch ($Action) {
    "Create" {
        $vms = Get-Content $VMListFile
        $snapshotName = "Pre-Patch-$(Get-Date -Format yyyyMMdd)-$TicketNumber"
        $results = @()

        foreach ($vmName in $vms) {
            try {
                $vm = Get-VM -Name $vmName -ErrorAction Stop
                # Check datastore space first
                $ds = Get-Datastore -VM $vm | Sort-Object FreeSpaceGB | Select-Object -First 1
                $freePercent = [math]::Round(($ds.FreeSpaceGB / $ds.CapacityGB) * 100, 1)

                if ($freePercent -lt 20) {
                    Write-Warning "SKIP $vmName - Datastore '$($ds.Name)' only $freePercent% free"
                    continue
                }

                New-Snapshot -VM $vm -Name $snapshotName -Description "Patching window - $TicketNumber" -Memory -Quiesce -ErrorAction Stop
                $results += [PSCustomObject]@{ VM = $vmName; Status = "Success"; Snapshot = $snapshotName }
                Write-Host "[OK] $vmName - Snapshot created" -ForegroundColor Green
            }
            catch {
                $results += [PSCustomObject]@{ VM = $vmName; Status = "FAILED"; Snapshot = $_.Exception.Message }
                Write-Host "[FAIL] $vmName - $($_.Exception.Message)" -ForegroundColor Red
            }
        }
        $results | Export-Csv "C:\patch-snapshot-report-$(Get-Date -Format yyyyMMdd).csv" -NoTypeInformation
    }

    "Remove" {
        $vms = Get-Content $VMListFile
        foreach ($vmName in $vms) {
            Get-VM $vmName | Get-Snapshot |
                Where-Object { $_.Name -like "Pre-Patch*" -and $_.Created -lt (Get-Date).AddDays(-$MaxAgeDays) } |
                ForEach-Object {
                    Write-Host "Removing: $($_.VM.Name) / $($_.Name) (Age: $([math]::Round(((Get-Date)-$_.Created).TotalDays,1))d)"
                    Remove-Snapshot -Snapshot $_ -Confirm:$false
                }
        }
    }

    "Report" {
        Get-VM | Get-Snapshot | Sort-Object Created |
            Select-Object VM, Name, Created,
                @{N="AgeDays";E={[math]::Round(((Get-Date)-$_.Created).TotalDays,1)}},
                @{N="SizeGB";E={[math]::Round($_.SizeGB,2)}} |
            Format-Table -AutoSize
    }

    "Revert" {
        $vms = Get-Content $VMListFile
        foreach ($vmName in $vms) {
            $snap = Get-VM $vmName | Get-Snapshot | Where-Object { $_.Name -like "Pre-Patch*" } | Select-Object -Last 1
            if ($snap) {
                Write-Host "Reverting $vmName to snapshot: $($snap.Name)"
                Set-VM -VM (Get-VM $vmName) -Snapshot $snap -Confirm:$false
            }
        }
    }
}
```

---


## 13. Troubleshooting

### 13.1 VM Won't Boot After Patch

| Symptom | Possible Cause | Resolution |
|---------|---------------|------------|
| Grub rescue prompt | Kernel package corruption | Revert snapshot, re-patch with `--exclude=kernel*` |
| Kernel panic | Incompatible kernel module | Revert snapshot; boot older kernel via GRUB |
| Hangs at boot | systemd service timeout | Connect via vCenter console, boot to rescue mode |
| Black screen | Display driver issue (rare on VMs) | Wait 5 min; if no progress, revert snapshot |

**Emergency Recovery (no snapshot):**
```bash
# From vCenter Console, interrupt GRUB:
# Press 'e' at GRUB menu
# Change 'linux' line: append 'init=/bin/bash' or 'rd.break'
# Press Ctrl+X to boot
# Remount root: mount -o remount,rw /
# Fix issue, then: exec /sbin/init
```

### 13.2 Snapshot Consolidation Issues

```bash
# Check consolidation status
govc vm.info "prod-web-01" | grep -i consolidat

# Force consolidation
govc snapshot.remove -vm "prod-web-01" -consolidate
```

```powershell
# Find VMs needing consolidation
Get-VM | Where-Object { $_.ExtensionData.Runtime.ConsolidationNeeded -eq $true } |
    Select-Object Name, PowerState

# Trigger consolidation
(Get-VM "prod-web-01").ExtensionData.ConsolidateVMDisks()
```

**If consolidation fails:**
1. Power off VM (schedule downtime)
2. Retry consolidation while powered off
3. If still failing, clone VM to new datastore and decommission original

### 13.3 VMware Tools Broken After Patch

```bash
# Reinstall open-vm-tools
sudo yum reinstall open-vm-tools -y    # RHEL 7
sudo dnf reinstall open-vm-tools -y    # RHEL 8/9/10

# Restart service
sudo systemctl restart vmtoolsd
sudo systemctl enable vmtoolsd

# Verify
vmware-toolbox-cmd -v
systemctl status vmtoolsd
```

### 13.4 Lost Network After Patch

| Cause | Fix |
|-------|-----|
| NetworkManager updated, config reset | `nmcli connection reload && nmcli connection up eth0` |
| vmxnet3 driver mismatch | Update open-vm-tools, reboot |
| Interface renamed (udev rules) | Check `ip link show`, update config for new name |
| firewalld rules reset | `firewall-cmd --reload` or restore rules |

```bash
# Emergency network restore via vCenter console
nmcli device status
nmcli connection show
nmcli connection up "System eth0"

# If interface renamed
ip link show
# Look for new interface name, update /etc/sysconfig/network-scripts/
```

### 13.5 Thick/Thin Disk Snapshot Issues

| Disk Type | Snapshot Behavior | Risk |
|-----------|-------------------|------|
| Thin provisioned | Snapshot delta is thin | Lower initial space usage |
| Thick Lazy Zeroed | Snapshot delta is thin | Delta grows with writes |
| Thick Eager Zeroed | Snapshot delta is thin | Delta still grows; FT disks can't snapshot |

> 💡 **Key Point**: Regardless of base disk type, snapshot deltas are always thin-provisioned. Monitor datastore space!

### 13.6 Datastore Full

```bash
# Emergency: Identify largest snapshot files
govc datastore.ls -l "Datastore-SAN-01" | sort -k1 -rn | head -20

# Find VMs with large snapshots
govc find / -type m -runtime.consolidationNeeded true
```

```powershell
# Find VMs consuming most snapshot space
Get-VM | Get-Snapshot | Sort-Object SizeGB -Descending |
    Select-Object -First 10 VM, Name, @{N="SizeGB";E={[math]::Round($_.SizeGB,2)}}, Created
```

**Emergency Actions:**
1. Delete non-critical snapshots immediately
2. Storage vMotion VMs to a datastore with free space
3. Expand datastore (add LUN/extent)
4. Never let datastore hit 100% — VMs will pause/crash

---

## 14. Best Practices & Common Mistakes

### ✅ Best Practices

| # | Practice | Why |
|---|----------|-----|
| 1 | Always snapshot before patching | Instant rollback capability |
| 2 | Delete snapshots within 24-72 hours | Prevent performance degradation |
| 3 | Use `open-vm-tools` from OS repos | VMware-recommended, auto-updated with OS |
| 4 | Check datastore space before snapshots | Prevent datastore exhaustion |
| 5 | Patch in waves (serial, not all at once) | Limit blast radius |
| 6 | Test patches in Dev/QA first | Catch issues before production |
| 7 | Document everything in change tickets | Audit trail and knowledge sharing |
| 8 | Verify VMware Tools after kernel update | Kernel modules may need rebuild |
| 9 | Use automation for consistency | Ansible/PowerCLI reduce human error |
| 10 | Schedule ESXi and guest patching separately | Different risk profiles and teams |

### ❌ Common Mistakes

| # | Mistake | Consequence |
|---|---------|-------------|
| 1 | Leaving snapshots for weeks | Datastore full, performance tanked, consolidation failure |
| 2 | Patching all VMs simultaneously | Total outage if patch is bad |
| 3 | Not checking datastore space first | Snapshot fills datastore, VMs crash |
| 4 | Ignoring VMware Tools status | Quiesced snapshots fail, no graceful shutdown |
| 5 | Confusing ESXi patching with guest patching | Wrong maintenance mode, unnecessary downtime |
| 6 | Forgetting to re-enable DRS | VMs not balanced, resource contention |
| 7 | Reverting snapshot after days of changes | Data loss for all post-snapshot writes |
| 8 | Not validating backups before patching | No safety net if snapshot also fails |
| 9 | Patching vCenter and ESXi same window | Lose management if something fails |
| 10 | Using memory snapshots on large VMs | Snapshot takes forever, huge files |

---

## 15. Quick Reference Commands

### Snapshot Operations

| Action | govc | PowerCLI |
|--------|------|----------|
| Create snapshot | `govc snapshot.create -vm "VM" -m "Name"` | `New-Snapshot -VM $vm -Name "Name" -Memory` |
| List snapshots | `govc snapshot.tree -vm "VM"` | `Get-VM "VM" \| Get-Snapshot` |
| Revert snapshot | `govc snapshot.revert -vm "VM" "Name"` | `Set-VM -VM $vm -Snapshot $snap` |
| Delete snapshot | `govc snapshot.remove -vm "VM" "Name"` | `Remove-Snapshot -Snapshot $snap` |
| Delete all | `govc snapshot.remove -vm "VM" "*"` | `Get-Snapshot -VM $vm \| Remove-Snapshot` |

### VM Information

| Action | govc | PowerCLI |
|--------|------|----------|
| VM info | `govc vm.info "VM"` | `Get-VM "VM"` |
| Tools status | `govc vm.info "VM" \| grep tool` | `(Get-VM "VM").Guest.ToolsStatus` |
| Power on | `govc vm.power -on "VM"` | `Start-VM "VM"` |
| Power off | `govc vm.power -off "VM"` | `Stop-VM "VM"` |
| Graceful shutdown | `govc vm.power -s "VM"` | `Shutdown-VMGuest "VM"` |

### Patching Commands

| RHEL Version | Update Command | Security Only |
|-------------|----------------|---------------|
| RHEL 6/7 | `yum update -y` | `yum update --security -y` |
| RHEL 8 | `dnf update -y` | `dnf update --security -y` |
| RHEL 9 | `dnf update -y` | `dnf update --security -y` |
| RHEL 10 | `dnf update -y` | `dnf update --security -y` |

### Quick Checks

```bash
# Reboot needed?
needs-restarting -r

# Current kernel
uname -r

# System health
systemctl is-system-running

# Failed services
systemctl --failed

# VMware Tools version
vmware-toolbox-cmd -v

# Recent packages installed
rpm -qa --last | head -10
```

---

## ⚠️ Disclaimer

> **This document is provided as-is for educational and operational reference purposes.** Always test procedures in a non-production environment before applying to production systems. The authors are not responsible for any data loss, downtime, or damage resulting from the use of commands or procedures described in this guide. Adapt all commands, scripts, and workflows to match your organization's specific environment, policies, and change management processes. Always follow your organization's approved change management procedures and obtain proper authorization before making changes to production systems.

---

*Document Version: 2.0 | Last Updated: June 2026 | Author: Infrastructure Team*
*Companion to: Linux_Patching_Master_Runbook.md*
