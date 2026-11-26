# VMware vSphere ESXi 8 Home Lab (Nested Virtualization)

A hands-on VMware ESXi 8.0 U3 lab built on a laptop using **nested virtualization**: ESXi runs as a VM inside VMware Workstation Pro, and Linux VMs run inside ESXi.

The lab covers the day-to-day work of an ESXi administrator: installation, storage, VMs, snapshots and backups, networking, resource controls, security, the CLI and monitoring. vCenter features are covered as theory in Phase 10.

## Contents

1. [Installation](#phase-1-installation)
2. [Storage](#phase-2-storage)
3. [Virtual machines](#phase-3-virtual-machines)
4. [Snapshots & backup](#phase-4-snapshots--backup)
5. [Networking](#phase-5-networking)
6. [Resource management](#phase-6-resource-management)
7. [Security](#phase-7-security)
8. [CLI](#phase-8-cli-ssh-esxcli-vim-cmd)
9. [Monitoring](#phase-9-monitoring)
10. [vCenter concepts](#phase-10-vcenter-concepts-theory)

---

## Lab environment

| Component | Details |
|---|---|
| Physical host | HP Laptop 15s, AMD Ryzen 5 3500U, 8 GB RAM, Windows 11 Home |
| Type-2 hypervisor | VMware Workstation Pro 26H1 (free) |
| Nested hypervisor | VMware ESXi 8.0 Update 3e (build 24677879, free edition) |
| ESXi VM | 2 vCPU, 4 GB RAM, 40 GB system disk + 30 GB data disk, NAT network |
| Guest VM | Alpine Linux 3.24 (`alpine-01`): 1 vCPU, 512 MB RAM, 4 GB thin disk |
| Management | ESXi Host Client (web UI), SSH + `esxcli` / `vim-cmd` |

### Topology

```
Windows 11 laptop
 └─ VMware Workstation Pro (NAT: VMnet8, 192.168.146.0/24)
     └─ ESXi-01 (vmk0: 192.168.146.128)
         ├─ vSwitch0 ── uplink vmnic0
         │   ├─ Management Network (vmk0)
         │   └─ VM Network ── alpine-01 (192.168.146.129, DHCP)
         └─ vSwitch1 ── no uplink (internal only)
             └─ Lab-VLAN10 (VLAN ID 10)
```

---

## Phase 1: Installation

- Downloaded the free ESXi 8.0U3e ISO from the Broadcom Support Portal (Free Software Downloads)
- Verified the ISO with an MD5 checksum (`certutil -hashfile <iso> MD5`)
- Created a nested ESXi VM in Workstation with **"Virtualize Intel VT-x/EPT or AMD-V/RVI"** enabled
- Installed ESXi, set the root password, and logged in to the Host Client

**Issues fixed**
- Broadcom "Build your profile" is blocked for non-corporate email domains → the free downloads area works without it
- Root password rejected → ESXi password rules: an uppercase letter at the start and a digit at the end don't count toward complexity
- Login failed → the username is case-sensitive (`root`, not `Root`)
- Host Client unreachable after a reboot → restarted the management network from the DCUI (F2)

![ESXi Host Client](01-install/1-esxi-host-client.png)

## Phase 2: Storage

- ESXi 8 used the whole 40 GB system disk for **OSDATA + BOOTBANK1/2**, leaving no space for VMs
- Added a 30 GB virtual disk and created **datastore1** (VMFS 6)
- Uploaded ISO files to the datastore with the Datastore browser

![datastore1 created](02-storage/1-datastore1-created.png)

## Phase 3: Virtual machines

- Created `alpine-01` (1 vCPU, 512 MB, 4 GB **thin-provisioned** disk), booted it from a datastore ISO
- Installed Alpine to disk with `setup-alpine` (sys mode), then disconnected the ISO
- Installed **open-vm-tools** (VMware's recommended tools for Linux), enabling graceful shutdown and guest info in the Host Client

**Issues fixed**
- VM stuck in the EFI Boot Manager → selected the CD-ROM manually
- Switching firmware to BIOS failed: *"ACPI motherboard layout requires EFI"*
- VM booted to PXE after install → `setup-disk` had not completed; reran the install

![ISO uploaded to datastore](03-virtual-machines/1-iso-upload.png)
![alpine-01 created](03-virtual-machines/2-vm-created.png)
![Network and ping test](03-virtual-machines/3-network-ping.png)
![VMware Tools running](03-virtual-machines/4-vmware-tools.png)

## Phase 4: Snapshots & backup

- Took snapshot `clean-install` (no memory, no quiesce)
- Created `/root/test.txt`, restored the snapshot, confirmed the file was gone
- Deleted the snapshot (consolidated into the base disk)
- Exported the VM as **OVF** (`.ovf` + `.vmdk` + `.mf`); the 4 GB thin disk exported as about 200 MB

> Snapshots are not backups. They depend on the base disk and grow over time. Use them for short-term rollback, and use exports or backup tools for real backups.

![Snapshot manager](04-snapshots/1-snapshot-manage.png)
![After restore: test file gone](04-snapshots/2-snapshot-restored.png)
![OVF export files](04-snapshots/3-ovf-export-files.png)

## Phase 5: Networking

- Reviewed port groups, vSwitches, physical NICs (vmnic0) and VMkernel NICs (vmk0)
- Created **vSwitch1** with no uplink (internal only) and port group **Lab-VLAN10** (VLAN 10)
- Moved `alpine-01` to Lab-VLAN10 while it was running:
  - DHCP failed (no DHCP server on the isolated switch)
  - Static IP `10.10.10.11/24` → `ping 8.8.8.8`: *Network unreachable* (isolation confirmed)
- Moved it back to VM Network → DHCP address returned, 0% packet loss
- Reviewed vSwitch security: promiscuous mode, MAC address changes, forged transmits (all Reject)

![Port groups](05-networking/1-port-groups.png)
![vSwitch1 topology](05-networking/2-vswitch1-topology.png)
![Isolated: no DHCP lease](05-networking/3-isolated-no-dhcp.png)
![Back on VM Network](05-networking/4-back-on-vm-network.png)

## Phase 6: Resource management

| Test | Result |
|---|---|
| CPU **limit** 200 MHz with a 100% CPU load (`yes > /dev/null`) | Held at ~210 MHz |
| Same load, no limit | ~1,500–2,300 MHz |
| Memory **reservation** 256 MB | VM swap file (`.vswp`) shrank from 512 MB to 256 MB |
| Admission control | Power-on refused when the host couldn't guarantee the reservation |

> Reservations guarantee resources, limits cap them, and shares set priority under contention (High = 2× Normal = 4× Low). Each VM has two swap files: the VM swap (`.vswp` = memory − reservation) and the VMX overhead swap (`vmx-*.vswp`).

![CPU limit setting](06-resources/1-cpu-limit-setting.png)
![CPU held at ~210 MHz](06-resources/2-cpu-limited-212mhz.png)
![.vswp at 256 MB](06-resources/3-memory-reservation-vswp.png)

## Phase 7: Security

- Created local user `viewer` (no shell access) and gave it the built-in **Read-only** role on the host
- Logged in as `viewer` in a private window:
  - No Manage menu
  - Power, edit, snapshot, export and delete actions all disabled
- Reviewed **lockdown mode** (left disabled):
  - **Normal**: management only through vCenter; DCUI still available
  - **Strict**: DCUI also disabled
  - **Exception users**: accounts that keep direct access (for example, backup service accounts)

![Local users](07-security/1-users.png)
![Logged in as viewer](07-security/2-viewer-login.png)
![Power actions blocked for viewer](07-security/3-viewer-power-blocked.png)
![Lockdown mode](07-security/4-lockdown-mode.png)

## Phase 8: CLI (SSH, esxcli, vim-cmd)

Enabled the **TSM-SSH** service, connected from PowerShell, and ran:

| Command | Shows |
|---|---|
| `esxcli system version get` | ESXi 8.0.3, build 24677879 |
| `esxcli hardware cpu global get` | CPU packages, cores, threads |
| `esxcli network ip interface ipv4 get` | vmk0 IP, netmask, gateway |
| `esxcli network vswitch standard list` | vSwitches, uplinks, port groups |
| `esxcli storage filesystem list` | datastore1 (VMFS-6), OSDATA, BOOTBANKs |
| `vim-cmd vmsvc/getallvms` | VM list with Vmid |
| `vim-cmd vmsvc/power.getstate 1` | Power state of alpine-01 |
| `vim-cmd vmsvc/power.shutdown 1` / `power.on 1` | Graceful shutdown / power on |

Stopped SSH afterwards (security best practice).

![esxcli: version, CPU, network](08-cli/1-esxcli-version-cpu-network.png)
![esxcli: vSwitches and storage](08-cli/2-esxcli-vswitch-storage.png)
![vim-cmd: VM list and power state](08-cli/3-vim-cmd-power.png)

## Phase 9: Monitoring

- **Host performance:** CPU ~7% average (peak ~50%: one vCPU on a two-core host), memory ~1.9 of 4 GB, low network and disk, disk latency max 5 ms
- **VM performance:** CPU load spikes up to 65%; **CPU Ready** max 5.8% (healthy; above ~10% indicates host contention)
- **Events and tasks:** full audit trail of lab actions; `vpxuser@127.0.0.1` logins are an internal service account
- **Logs:** `hostd.log` (management and Host Client logins), `vmkernel.log` (storage, network, hardware), `auth.log` (SSH); ESXi logs use **UTC** time; *Generate support bundle* collects logs for VMware support
- **Notifications:** none, which means a healthy host

![Host CPU](09-monitoring/1-host-cpu.png)
![Host memory](09-monitoring/2-host-memory.png)
![Host network](09-monitoring/3-host-network.png)
![Host disk](09-monitoring/4-host-disk.png)
![VM CPU and CPU Ready](09-monitoring/5-vm-cpu-ready.png)
![Events](09-monitoring/6-events.png)
![Tasks](09-monitoring/7-tasks.png)
![auth.log](09-monitoring/8-auth-log.png)

---

## Phase 10: vCenter concepts (theory)

vCenter isn't free and needs about 14 GB RAM, so these features are documented as theory.

| Feature | What it does | Requirements |
|---|---|---|
| **vCenter Server (VCSA)** | Central management of many ESXi hosts through the vSphere Client | Appliance deployed as a VM |
| **Datacenter / Cluster** | Logical grouping; a cluster pools hosts' CPU and memory | vCenter |
| **vMotion** | Moves a running VM to another host with no downtime | Shared storage (or combined vMotion), a vMotion VMkernel port, compatible CPUs |
| **Storage vMotion** | Moves a running VM's disks to another datastore | vCenter |
| **HA (High Availability)** | Restarts VMs on surviving hosts after a host failure (brief downtime) | Cluster, shared storage, network heartbeats |
| **FT (Fault Tolerance)** | Keeps a live shadow VM on another host; zero downtime | FT logging network; vCPU limits |
| **DRS** | Balances VMs across hosts automatically using vMotion | Cluster with DRS enabled |
| **Templates & cloning** | Build one golden VM, deploy many copies | vCenter |
| **vSphere Distributed Switch (vDS)** | One network configuration shared by all hosts | vCenter |
| **Enhanced vMotion Compatibility (EVC)** | Masks CPU features so vMotion works across CPU generations | Cluster setting |

### Key differences

- **HA vs FT:** HA restarts the VM (minutes of downtime); FT keeps it running (no downtime).
- **HA vs DRS:** HA reacts to failures; DRS balances load during normal operation.
- **Standard vSwitch vs vDS:** a standard vSwitch is configured on each host separately; a vDS is configured once in vCenter.
- **Snapshot vs backup:** a snapshot depends on the base disk; a backup is an independent copy.

---

## Skills demonstrated

ESXi installation and troubleshooting · VMFS datastores · VM lifecycle and VMware Tools · snapshots and OVF export · vSwitches, port groups and VLANs · reservations, limits and admission control · RBAC and lockdown mode · `esxcli` and `vim-cmd` · performance monitoring (CPU Ready, latency) and log analysis

## Repository structure

```
vsphere-esxi8-homelab/
├── README.md
├── 01-install/
├── 02-storage/
├── 03-virtual-machines/
├── 04-snapshots/
├── 05-networking/
├── 06-resources/
├── 07-security/
├── 08-cli/
└── 09-monitoring/
```

Each folder holds that phase's screenshots, numbered in the order the steps were done. Phase 10 is theory only, so it has no folder.

## Author

**Mohammed Junaid** · [LinkedIn](https://www.linkedin.com/in/mohammed-junaid-221b7323b) · [GitHub](https://github.com/MohammedJunaidspec)
