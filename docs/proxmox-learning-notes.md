# Proxmox VE Learning Notes

> **Purpose:** A living reference for the Proxmox VE concepts,
> configuration details, and commands learned during hands-on practice.
> We will update this file as the learning progresses.
>
> **Scope:** Proxmox VE only. The future pfSense/NGFW project is
> intentionally out of scope for this document.

## 1. Lab environment

  Item                         Current details
  ---------------------------- ------------------------------------------------------
  Proxmox VE version           9.2.2
  Deployment                   Proxmox VE running as a VM inside VMware Workstation
  VMware Workstation version   26.0.0.25388281
  Proxmox vCPU                 2
  Proxmox RAM                  Approximately 3.8 GiB (about 4 GB allocated)
  Proxmox virtual disk         20 GB
  Proxmox management IP        `192.0.2.100/24`
  Default gateway              `192.0.2.1`
  Proxmox network interface    `nic0`
  Linux bridge                 `vmbr0`
  VMware network               VMnet8 NAT, subnet `192.0.2.0/24`

**Important:** Proxmox is nested inside VMware Workstation. Its
available CPU, memory, and disk are limited by the resources assigned to
its VMware VM. The laptop's total 16 GB RAM is not all available to
Proxmox.

### Other systems in the wider lab

These systems exist in the wider learning environment, but are not
Proxmox components:

-   Windows Server 2025 VM in VMware Workstation: 2 vCPU, 2 GB RAM, 60
    GB disk.
-   Windows client VM in VMware Workstation.
-   Kali Linux through WSL on the Windows host.
-   Local AI models through Ollama on the Windows host.

## 2. What is Proxmox VE?

**Proxmox Virtual Environment (VE)** is an open-source virtualization
platform based on Debian Linux. It provides a web interface and
command-line tools to manage virtual machines, Linux containers,
storage, networking, backups, and other virtualization resources.

### Core terminology

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  Datacenter                          Top-level view for managing Proxmox
                                      resources. In a single-node lab, it
                                      contains the node and its
                                      resources.

  Node                                A Proxmox host. Our node is named
                                      `proxmox`.

  VM (Virtual Machine)                A virtual computer with virtual
                                      CPU, memory, disk, and network
                                      devices.

  LXC container                       A lightweight Linux container that
                                      shares the host's Linux kernel.

  Storage                             A configured location for VM disks,
                                      ISO images, backups, templates, and
                                      other supported content.

  Linux bridge                        A virtual Layer 2 switch that
                                      connects network interfaces,
                                      including VM interfaces.

  ISO image                           Installation media used to install
                                      an operating system in a VM.

  Snapshot                            A point-in-time state of a VM or
                                      its disks, depending on
                                      configuration and storage support.
                                      It is not a substitute for an
                                      independent backup.

  Backup                              A recoverable copy of a VM or
                                      container.

  Template                            A prepared base image used to
                                      create similar VMs or containers.

  vCPU                                A virtual CPU presented to a VM.

  LVM                                 Logical Volume Manager, used to
                                      manage block storage volumes.

  LVM-thin                            Thin-provisioned storage commonly
                                      used for VM disks and container
                                      volumes.
  -----------------------------------------------------------------------

## 3. Proxmox web interface

Proxmox is commonly administered through a browser.

-   Open the Proxmox VM console in VMware and note the management IP
    shown during startup, or inspect it using `ip -br addr`.
-   From the Windows host, browse to `https://192.0.2.100:8006/`
    while the host can reach that network.
-   Port `8006` is the Proxmox web management interface.

Useful areas in the interface:

  Area               Purpose
  ------------------ ---------------------------------------------------
  Datacenter         Top-level resource and configuration view
  Node (`proxmox`)   Host-level configuration and status
  Summary            Resource usage and system information
  Shell              Terminal access to the Proxmox host
  Network            Network interfaces and Linux bridge configuration
  Storage            Storage definitions, capacity, and content
  Backup             Backup configuration and available backups
  Permissions        Users, groups, roles, and access control

## 4. Current Proxmox network configuration

The network configuration inspected during practice was:

``` text
auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.0.2.100/24
    gateway 192.0.2.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0

source /etc/network/interfaces.d/*
```

### Explanation

-   `lo`: Loopback interface, used for local communication within the
    system.
-   `nic0`: Network interface presented to the Proxmox VM by VMware. It
    is configured as `manual` because the IP configuration is assigned
    to the bridge.
-   `vmbr0`: Linux bridge. It acts like a virtual network switch.
-   `address 192.0.2.100/24`: Static management IP assigned to
    `vmbr0`.
-   `gateway 192.0.2.1`: Default gateway, identified in this lab as
    the VMware NAT gateway.
-   `bridge-ports nic0`: Connects `nic0` to `vmbr0`.
-   `bridge-stp off`: Disables Spanning Tree Protocol on this bridge.
-   `bridge-fd 0`: Sets the bridge forwarding delay to zero.
-   `source /etc/network/interfaces.d/*`: Includes additional interface
    configuration files from that directory.

### Current network path

``` text
VMware VMnet8 (NAT)
  Subnet: 192.0.2.0/24
          |
        nic0
          |
        vmbr0
  Proxmox IP: 192.0.2.100/24
  Gateway:    192.0.2.1
```

**Safety note:** `vmbr0` currently provides Proxmox management
connectivity. Do not change its IP, gateway, bridge port, or network
configuration casually. A mistake can disconnect the Proxmox web
interface. Inspect and plan network changes before applying them.

## 5. Storage observed in this lab

The following storage was shown by `pvesm status`:

  --------------------------------------------------------------------------------
  Storage ID    Type             Observed Observed used      Observed Typical
                                    total                   available purpose
  ------------- ----------- ------------- ------------- ------------- ------------
  `local`       Directory          8.3 GB        4.1 GB        3.7 GB Commonly ISO
                                                                      images,
                                                                      backups,
                                                                      templates,
                                                                      and other
                                                                      file-based
                                                                      content

  `local-lvm`   LVM-thin           6.5 GB          0 GB        6.5 GB Commonly VM
                                                                      disks and
                                                                      container
                                                                      volumes
  --------------------------------------------------------------------------------

Values are from the earlier lab inspection and may change as storage is
used. The configured content types can be checked in the storage
configuration and web interface.

### Disk layout observed

The `lsblk` output showed a 20 GB virtual disk named `sda`, with
partitions and LVM volumes including:

-   `pve-root`: approximately 8.6 GB --- Proxmox root filesystem.
-   `pve-swap`: approximately 2.4 GB --- swap space.
-   `pve-data`: approximately 6.6 GB --- thin-pool storage.

These are the values observed in this lab, not universal Proxmox
defaults.

## 6. Resource observations

The earlier `free -h` output showed approximately:

  Metric               Observed value
  ------------------ ----------------
  Total memory                3.8 GiB
  Used memory                 1.6 GiB
  Free memory                 1.9 GiB
  Buff/cache                  523 MiB
  Available memory            2.2 GiB
  Swap total                  2.4 GiB

Interpretation:

-   **Total** is the memory visible to Proxmox.
-   **Used** is memory currently in use.
-   **Free** is completely unused memory.
-   **Buff/cache** is memory used for filesystem buffers and cache.
-   **Available** estimates memory that can be made available to
    applications without swapping heavily.
-   **Swap** is disk-backed memory space; it is not equivalent to
    physical RAM.

The Proxmox dashboard previously showed 2 vCPUs. The Proxmox VM
currently has approximately 4 GB RAM allocated, not the host laptop's
entire 16 GB.

## 7. Commands learned so far

All commands below were used for inspection. They are read-only unless
otherwise noted.

### System and version

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `pveversion -v`                     Displays Proxmox VE and installed
                                      package versions.

  `cat /etc/os-release`               Displays the Linux distribution and
                                      release information.

  `hostname -I`                       Displays IP addresses assigned to
                                      the host. In our lab it returned
                                      `192.0.2.100`.

  `lscpu`                             Displays CPU architecture, CPU
                                      count, model, and related CPU
                                      information.
  -----------------------------------------------------------------------

Useful filtered CPU view:

``` bash
lscpu | grep -E 'CPU\(s\)|Model name|Virtualization'
```

### Memory and disk

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `free -h`                           Shows RAM and swap usage in
                                      human-readable units.

  `lsblk`                             Lists block devices, partitions,
                                      and logical volumes.

  `pvesm status`                      Shows configured Proxmox storage,
                                      type, capacity, usage, and
                                      availability.

  `cat /etc/pve/storage.cfg`          Displays Proxmox storage
                                      definitions.
  -----------------------------------------------------------------------

### Networking

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `ip -br addr`                       Brief view of network interfaces
                                      and their IP addresses.

  `ip -br link`                       Brief view of network interfaces
                                      and link state.

  `ip route`                          Displays the routing table,
                                      including the default gateway.

  `cat /etc/network/interfaces`       Displays the main network interface
                                      and bridge configuration file.

  `bridge link`                       Shows interfaces attached to Linux
                                      bridges.

  `bridge vlan show`                  Displays VLAN configuration
                                      associated with bridge ports.

  `ip -d link show type bridge`       Displays detailed information about
                                      Linux bridge interfaces.

  `ip -br link \| grep tap`           Checks for TAP interfaces. No TAP
                                      interfaces may appear before VMs
                                      with virtual network adapters are
                                      running.

  `ping -c 4 192.0.2.1`            Tests connectivity to the VMware
                                      NAT gateway.

  `ping -c 4 8.8.8.8`                 Tests external IP connectivity. A
                                      successful ping confirms IP
                                      reachability, not necessarily DNS
                                      resolution.
  -----------------------------------------------------------------------

### Proxmox API and resource inventory

  ------------------------------------------------------------------------
  Command                              Purpose
  ------------------------------------ -----------------------------------
  `pvesh get /nodes`                   Lists Proxmox nodes through the
                                       Proxmox API command-line interface.

  `pvesh get /nodes/proxmox/network`   Displays network configuration
                                       reported by the Proxmox API for
                                       node `proxmox`.
  ------------------------------------------------------------------------

### Command notes

-   `-h` means human-readable output for commands such as `free`.
-   `-c 4` makes `ping` send four packets.
-   `-br` requests brief output from `ip`.
-   `-d` requests detailed link information.
-   `grep` filters output to matching text.
-   `cat` prints file contents.
-   `|` passes one command's output to another command.

## 8. Connectivity checks completed

The following tests succeeded:

``` bash
ping -c 4 192.0.2.1
ping -c 4 8.8.8.8
hostname -I
```

Results:

-   Proxmox could reach `192.0.2.1`, the VMware NAT gateway.
-   Proxmox could reach `8.8.8.8` by IP.
-   `hostname -I` returned `192.0.2.100`.

This confirmed basic gateway and external IP connectivity at the time of
testing.

## 9. Important precautions

1.  Do not modify `/etc/network/interfaces` until you understand the
    bridge, address, gateway, and bridge-port relationships.
2.  Do not restart networking or apply bridge changes remotely without a
    recovery plan and access to the VMware console.
3.  Do not assume that a snapshot is a backup.
4.  Monitor Proxmox memory and disk capacity before creating VMs.
5.  Remember that Proxmox is nested inside VMware in this lab. Resource
    allocations are constrained by both the physical host and the VMware
    VM settings.
6.  Keep the current `vmbr0` management network working while learning
    additional bridges.
7.  Prefer inspection commands before configuration-changing commands.

## 10. Learning progress tracker

-   [x] Identify Proxmox VE version and dashboard.
-   [x] Understand Datacenter and Node at an introductory level.
-   [x] Inspect memory, swap, disks, storage, interfaces, routes, and bridges.
-   [x] Test gateway and external IP connectivity.
-   [x] Review storage types and download a Debian installation ISO.
-   [x] Create Debian VM 100 and inspect its hardware configuration.
-   [x] Learn VM lifecycle commands and inspect VM configuration.
-   [x] Understand full clones, linked clones, templates, and snapshots.
-   [x] Inspect Linux bridge and virtual NIC configuration.
-   [x] Create isolated bridge `vmbr1`.
-   [x] Enable VLAN awareness on `vmbr1` and add persistent VLAN settings.
-   [x] Review Proxmox firewall levels and CLI/configuration references.
-   [x] Review backup, scheduling, retention, restore, and snapshot commands conceptually.
-   [x] Review template creation, cloning, and Cloud-Init workflows conceptually.
-   [ ] Resolve nested KVM availability before starting VMs.
-   [ ] Install and boot Debian VM 100.
-   [ ] Attach VM interfaces to VLAN 10 and VLAN 20 and test connectivity.
-   [ ] Implement and test actual Proxmox firewall rules.
-   [ ] Perform an actual backup and restore exercise.
-   [ ] Create and test a Cloud-Init template.
-   [ ] Learn LXC containers.
-   [ ] Learn users, roles, and permissions.
-   [ ] Practice troubleshooting and resource management.

## 11. How we will maintain this document

As each new Proxmox topic is completed, update this file with:

1.  **Concept:** What the feature is and why it is used.
2.  **Where to find it:** Relevant Proxmox UI location.
3.  **Commands:** Commands used, with explanations and examples.
4.  **Lab configuration:** Actual values from this environment.
5.  **Procedure:** Steps performed during the exercise.
6.  **Troubleshooting:** Errors, causes, and verified solutions.
7.  **Safety notes:** Any risks before changing configuration.
8.  **Progress:** Update the learning checklist.

> **Current stopping point:** Proxmox fundamentals, VM configuration,
> bridges, VLAN-aware networking, firewall concepts, backup/restore
> commands, and template workflows have been discussed. The isolated
> bridge and VLAN-aware setting have been configured, but VM-level VLAN
> assignments and connectivity tests remain pending. VM 100 cannot
> currently start because nested KVM virtualization is unavailable.


## 12. VM lifecycle, hardware, cloning, and snapshots

### VM 100: Debian lab VM

The VM was created with these observed settings:

| Setting | Value |
|---|---|
| VM ID | `100` |
| Name | `debian-lab` |
| BIOS | SeaBIOS |
| Machine | i440fx |
| CPU | 1 socket, 1 core |
| Memory | 1024 MiB |
| Disk | 12 GB on `local-lvm` |
| SCSI controller | VirtIO SCSI single |
| ISO | Debian 13.7 Netinst |
| Network | VirtIO NIC, initially connected to `vmbr0` |
| Firewall integration | `firewall=1` on the NIC |

Useful commands:

```bash
qm list
qm status 100
qm config 100
```

- `qm list`: lists VMs and their state.
- `qm status 100`: reports the state of VM 100.
- `qm config 100`: displays its hardware and configuration settings.

The configuration included a disk on `scsi0`, installation media on `ide2`,
and a virtual NIC on `net0`.

### VM lifecycle commands

```bash
qm start 100
qm shutdown 100
qm stop 100
qm reboot 100
qm reset 100
```

These commands respectively start, request a graceful shutdown, force-stop,
reboot, and reset a VM. `qm stop` and `qm reset` can cause guest data loss
if used while the guest is writing data.

### Clone, template, and snapshot

- **Clone:** Creates another VM based on an existing VM or template.
- **Template:** A reusable master image from which VMs can be cloned.
- **Snapshot:** Captures a point-in-time VM state for rollback; it is not an
  independent backup.

```bash
# Full clone: independent copy of the source disks
qm clone 100 101 --name debian-server --full 1

# Linked clone: depends on the source template/base image
qm clone 100 102 --name debian-client --full 0

# Convert a stopped VM into a template
qm template 100

# Snapshot operations
qm snapshot 100 before-update
qm listsnapshot 100
qm rollback 100 before-update
qm delsnapshot 100 before-update
```

A VM generally needs to be stopped before conversion to a template.
Linked clones require a suitable template/base and compatible storage.

## 13. Nested virtualization issue

When attempting to start VM 100, Proxmox returned:

```text
KVM virtualisation configured, but not available.
Either disable in VM configuration or enable in BIOS.
```

The Proxmox instance itself is running as a VM inside VMware Workstation.
For KVM acceleration inside Proxmox, nested Intel VT-x/EPT must be exposed
through the outer virtualization layer and available to the guest.

The VMware setting `Virtualize Intel VT-x/EPT` produced a warning that
virtualized Intel VT-x/EPT was not supported on the platform. Windows
diagnostics also reported a hypervisor detected, VBS running, and Memory
Integrity enabled.

**Status:** The nested virtualization issue has not been resolved. No
Windows security settings should be disabled without considering the
security impact. Do not assume that the Debian VM can start until KVM
availability is verified.

Useful diagnostic commands:

```bash
# On Proxmox
egrep -c '(vmx|svm)' /proc/cpuinfo
ls -l /dev/kvm
```

If `/dev/kvm` is absent or hardware virtualization flags are not exposed,
VM startup with KVM acceleration will fail. These checks are diagnostic;
they do not themselves enable nested virtualization.

## 14. Linux bridges and isolated networking

### Existing bridge: vmbr0

`vmbr0` is the management/upstream bridge and must be preserved. Its
configuration is documented in Section 4.

### New isolated bridge: vmbr1

The following configuration was added below the `vmbr0` section in
`/etc/network/interfaces`:

```text
auto vmbr1
iface vmbr1 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10 20
```

Meaning:

- `auto vmbr1`: bring the bridge up automatically.
- `inet manual`: do not assign an IP address to the bridge.
- `bridge-ports none`: no physical-facing interface is attached.
- `bridge-stp off`: disable Spanning Tree Protocol.
- `bridge-fd 0`: set forwarding delay to zero.
- `bridge-vlan-aware yes`: enable VLAN-aware bridge operation.
- `bridge-vids 10 20`: permit VLAN IDs 10 and 20 on the bridge.

The bridge was created with:

```bash
ifup vmbr1
```

It was inspected with:

```bash
ip link show vmbr1
ip -br link
ip -d link show vmbr1
bridge link
bridge vlan show
```

Runtime VLAN filtering was enabled with:

```bash
ip link set dev vmbr1 type bridge vlan_filtering 1
```

The detailed link output subsequently showed:

```text
vlan_filtering 1
vlan_protocol 802.1Q
```

The persistent configuration uses `bridge-vlan-aware yes` and
`bridge-vids 10 20`. The runtime command alone would not survive a reboot;
the persistent interface configuration is what makes the setting durable
when the network configuration is brought up normally.

**Important distinction:** `bridge-vids 10 20` permits those VLAN IDs on
the bridge. It does not by itself create VM interfaces, assign VMs to
VLANs, configure IP addresses, or provide inter-VLAN routing.

### Intended VLAN layout

| VLAN ID | Name | Intended use |
|---|---|---|
| 10 | `LAB-SERVERS` | Server VMs |
| 20 | `LAB-CLIENTS` | Client VMs |

A VM NIC configured with `tag=10` is placed in VLAN 10; `tag=20` places
it in VLAN 20.

Example VM NIC configuration (reference only):

```bash
# Example: set VM 100 NIC to VLAN 10 on vmbr1
qm set 100 --net0 virtio=02:00:00:00:00:10,bridge=vmbr1,tag=10,firewall=1

# Example: set VM 101 NIC to VLAN 20 on vmbr1
qm set 101 --net0 virtio,bridge=vmbr1,tag=20,firewall=1
```

The first command uses the MAC address observed for VM 100. **These
commands were examples; the user did not confirm executing them.** VM 100
has not been moved from `vmbr0` to `vmbr1`.

After editing `/etc/network/interfaces`, inspect the file and plan the
change before applying it. `ifreload -a` is commonly used to apply
ifupdown2 network configuration, but a network reload can disrupt
connectivity. Keep VMware console access available and do not alter
`vmbr0` during this exercise.

## 15. Proxmox firewall: architecture and CLI reference

Proxmox firewall controls exist at three scopes:

1. Datacenter/cluster level.
2. Node/host level.
3. Individual VM or container level.

The Proxmox firewall service can be running while firewall enforcement is
disabled. The `firewall=1` option on a VM NIC enables firewall integration
for that NIC; it does not, by itself, mean that all firewall levels and
policies are enabled.

Commands discussed:

```bash
pve-firewall status
systemctl status pve-firewall --no-pager
pve-firewall compile
pve-firewall localnet

pvesh get /cluster/firewall/options
pvesh get /nodes/proxmox/firewall/options

qm config 100 | grep firewall
qm config 100 | grep net
```

Configuration locations discussed:

```text
/etc/pve/datacenter.cfg
/etc/pve/firewall/cluster.fw
/etc/pve/nodes/proxmox/host.fw
/etc/pve/firewall/100.fw
```

Example VM firewall file (illustrative; validate syntax and policy for
the installed Proxmox release before applying):

```ini
[OPTIONS]
enable: 1

[RULES]
IN ACCEPT -p tcp -dport 22
IN ACCEPT -p tcp -dport 80
IN DROP -p tcp -dport 23
```

The example allows inbound TCP 22 and 80 and drops inbound TCP 23. It is
not a complete production firewall policy. Rules, default policies,
Datacenter/Node/VM enablement, and NIC firewall integration must be
considered together.

**Lab status:** Firewall architecture and command references were
discussed. No firewall rules were confirmed as created or tested. The
observed firewall service was active, while `pve-firewall status`
reported the firewall as disabled. The `cluster.fw` and `host.fw` files
were not present at the time of inspection.

Avoid changing default policies or enabling broad filtering on the
management interface without first allowing required management traffic
and ensuring console access.

## 16. Backup, scheduling, retention, restore, and snapshots

### Backup with vzdump

`vzdump` creates backups of Proxmox VMs and containers.

```bash
# Basic syntax
vzdump <VMID> --storage <storage> --mode <mode>

# Examples
vzdump 100 --storage local --mode stop
vzdump 100 --storage local --mode snapshot
vzdump 100 --storage local --mode suspend

# Zstandard compression
vzdump 100 --storage local --compress zstd
```

Modes:

- `stop`: stops the guest for the backup operation.
- `suspend`: uses a suspend-based backup approach.
- `snapshot`: uses snapshot-based backup where supported.

Backup files for the `local` directory storage are commonly found in:

```text
/var/lib/vz/dump/
```

List backup content:

```bash
pvesm list local --content backup
ls -lh /var/lib/vz/dump/
```

### Scheduled backups

Backup job configuration is stored in Proxmox's cluster configuration.
Inspect jobs with:

```bash
pvesh get /cluster/backup
cat /etc/pve/jobs.cfg
```

An example scheduled job can be created through the API CLI:

```bash
pvesh create /cluster/backup \
  --storage local \
  --schedule '02:00' \
  --all 1 \
  --mode snapshot \
  --enabled 1 \
  --prune-backups 'keep-last=7'
```

This is an example only. Check the API parameters supported by the installed
Proxmox release and select a suitable storage and guest selection before
creating a job.

Common retention properties:

```text
keep-last=7
keep-daily=7
keep-weekly=4
keep-monthly=12
keep-yearly=2
```

These can be combined in a prune-backups policy, for example:

```text
keep-last=3,keep-daily=7,keep-weekly=4,keep-monthly=12
```

### Restore using qmrestore

```bash
qmrestore <backup-file> <new-VMID> --storage <target-storage>
```

Example:

```bash
qmrestore /var/lib/vz/dump/<actual-backup-file>.vma.zst 101 --storage local-lvm
```

The backup filename must be replaced with the exact file that exists.
The destination VM ID must be available, and the destination storage must
have sufficient capacity and support the required VM disk content.

Other useful commands:

```bash
pvesm list local --content backup
ls -lh /var/lib/vz/dump/
qm config 101
qm status 101
```

### Snapshot commands

```bash
qm snapshot 100 before-update
qm listsnapshot 100
qm rollback 100 before-update
qm delsnapshot 100 before-update
```

**Snapshot vs backup:**

- A snapshot is a rollback point and is commonly dependent on the
  original VM storage.
- A backup is a separate recoverable archive that can be stored on
  another configured storage target.
- A snapshot is not a substitute for an independent backup.

**Lab status:** These backup, retention, restore, and snapshot commands
were discussed as reference material. The user explicitly chose not to
run a backup or restore. No backup was created. At the time of inspection,
`local` had approximately 3.13 GiB free, so capacity must be checked before
any future backup.

## 17. Templates, cloning, and Cloud-Init

### Convert an existing VM to a template

Prepare and shut down the VM first. Then:

```bash
qm shutdown 100
qm template 100
```

`qm template` converts the VM into a reusable template. Do not convert
the only working VM unless it is intended to become the base image.

### Clone a template

```bash
# Full clone
qm clone 100 101 --name debian-server --full 1

# Linked clone
qm clone 100 102 --name debian-client --full 0
```

A full clone copies the virtual disks and is independent of the source
template. A linked clone shares a base image and depends on that base
remaining available. Linked clones require compatible storage and a
template suitable for linked cloning.

### Cloud-Init template workflow (reference)

The following is an example workflow for a Debian cloud image. It is
not a procedure already performed in this lab. Confirm the image URL,
storage content types, free capacity, and actual imported volume ID
before using it.

```bash
# Example: download a Debian cloud image
mkdir -p /var/lib/vz/template/iso
cd /var/lib/vz/template/iso
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2

# Create an empty VM
qm create 200 --name debian-template --memory 2048 --cores 2 \
  --net0 virtio,bridge=vmbr1

# Import the cloud disk into LVM-thin storage
qm disk import 200 \
  /var/lib/vz/template/iso/debian-12-genericcloud-amd64.qcow2 \
  local-lvm

# Inspect the imported volume ID before attaching it
qm config 200

# Example only: replace the volume ID if the import output differs
qm set 200 --scsihw virtio-scsi-single \
  --scsi0 local-lvm:vm-200-disk-0

# Add Cloud-Init drive
qm set 200 --ide2 local-lvm:cloudinit

# Configure boot and serial console
qm set 200 --boot order=scsi0
qm set 200 --serial0 socket --vga serial0

# Configure initial Cloud-Init user and DHCP
qm set 200 --ciuser debian
qm set 200 --ipconfig0 ip=dhcp

# Convert to template after verifying the VM configuration
qm template 200

# Clone from the template
qm clone 200 201 --name debian-server --full 1
qm clone 200 202 --name debian-client --full 0
```

Cloud-Init requires a compatible cloud image and an attached Cloud-Init
drive. Additional settings such as SSH keys, DNS, static addressing, and
guest agent configuration can be added as needed. The imported volume ID
must be checked rather than assumed.

### Other template/VM administration commands

```bash
qm list
qm config 200
qm set 201 --memory 2048
qm set 201 --cores 2
qm resize 201 scsi0 +10G
qm destroy 201
```

`qm destroy` is destructive. Verify the VM ID and backup requirements
before using it. Disk resizing increases the virtual disk size; guest
partition/filesystem expansion may also be required.

## 18. Storage and resource cautions for this lab

At the time of the storage inspection:

- `local` had approximately 3.13 GiB available.
- `local-lvm` had approximately 6.55 GiB available.
- The Proxmox virtual disk was only 20 GB.
- Proxmox had approximately 3.8 GiB RAM allocated.

These values are time-sensitive and should be rechecked with:

```bash
pvesm status
df -h
free -h
lvs
```

Do not assume a 12 GB virtual disk requires a 12 GB backup; actual backup
size depends on used data and backup settings. Conversely, do not assume
it will fit in the available storage. Avoid importing large cloud images,
creating multiple clones, or running multiple VMs without checking space
and RAM first.

## 19. Current state and what has / has not been changed

### Confirmed changes

- `vmbr1` was added to `/etc/network/interfaces`.
- `vmbr1` was brought up using `ifup vmbr1`.
- Runtime VLAN filtering on `vmbr1` was enabled.
- Persistent VLAN-aware settings and permitted VLAN IDs 10 and 20 were
  added to the `vmbr1` configuration.

### Not confirmed as performed

- VM 100 was **not** confirmed as moved from `vmbr0` to `vmbr1`.
- VLAN tags were **not** confirmed as assigned to VM NICs.
- VLAN 10/20 IP addressing and connectivity tests were **not** performed.
- No Proxmox firewall rules were confirmed as created or tested.
- No backup or restore was performed.
- No template conversion, Cloud-Init setup, or clone operation was
  confirmed as performed.
- The nested KVM issue remains unresolved, so VM 100 cannot currently
  start.
