# LXC Containers in Proxmox VE

> **Purpose:** Concept reference for LXC containers in Proxmox VE. Pair every
> section with the matching lab in [`labs/lxc/`](../labs/lxc/) and record
> evidence under [`evidence/lxc/`](../evidence/lxc/).
>
> **Status:** Reference material. Lab execution status is tracked in the
> [lab index](../labs/README.md), not here.

Every topic in this document follows the same three-part pattern:

1. **Concept** — what it is and why it exists.
2. **Practical implementation** — the commands and configuration involved.
3. **Evidence and troubleshooting** — what to capture and what usually breaks.

---

## 1. LXC containers — architecture, features, use cases

### Concept

**LXC** (Linux Containers) is an operating-system-level virtualization
technology. It uses two kernel features to isolate processes:

- **Namespaces** — isolate what a process can *see* (PID, network, mount,
  IPC, UTS, user, cgroup, time).
- **Control groups (cgroups)** — limit what a process can *use* (CPU, memory,
  block I/O, PIDs).

There is **no hypervisor and no guest kernel**. Every container shares the
Proxmox host kernel. Proxmox VE wraps LXC in the `pct` tool ("Proxmox
Container Toolkit") and stores each container definition in
`/etc/pve/lxc/<CTID>.conf`.

```text
Proxmox VE host (Debian + KVM + LXC)
├── VM 100        -> own kernel, emulated hardware
├── CT 200        -> shares host kernel, isolated namespaces + cgroups
└── CT 201        -> shares host kernel, isolated namespaces + cgroups
```

### Features

| Feature | Detail |
|---|---|
| Boot time | Seconds — no kernel to boot |
| Resource overhead | Very low; no duplicate kernel or emulated devices |
| Density | Dozens of containers on modest hardware |
| Storage | Root filesystem is a directory/volume, not a disk image |
| Networking | `veth` pair attached to a Linux bridge (or none) |
| Console | `pct enter`, `pct exec`, or the web shell |

### Use cases

- Lightweight Linux services: web servers, reverse proxies, DNS, monitoring.
- Lab environments where density and speed matter more than isolation.
- Development sandboxes and CI build agents.
- Anything that does **not** need a non-Linux kernel, custom kernel modules,
  or full hardware emulation.

### Limitations

- **Linux only.** A container cannot run Windows or any non-Linux kernel.
- **Kernel version is fixed by the host.** No per-container kernel upgrades.
- **Weaker isolation than a VM.** A kernel exploit is a host-level risk.
- **Architecture bound.** An `amd64` host can only run `amd64` templates.

---

## 2. LXC vs VM

| Aspect | LXC container | KVM virtual machine |
|---|---|---|
| Kernel | Shared with host | Own, independent |
| Hardware | None emulated | Emulated/virtio devices |
| Boot time | ~1–3 s | ~10–60 s |
| Memory overhead | Minimal per container | Full guest RAM + QEMU overhead |
| Disk format | Directory or thin volume | Raw/qcow2 disk image |
| Max practical density | Very high | Limited by RAM |
| Isolation | Namespaces + cgroups + AppArmor | Hardware-assisted (VT-x/AMD-V) |
| Non-Linux guests | Not possible | Possible |
| Nested virtualization | Needs `nesting=1` (and host support) | Needs VT-x passthrough |
| Live migration | Yes, within a cluster | Yes, within a cluster |
| Snapshots | Yes, on snapshot-capable storage | Yes |
| Typical use | Linux services, labs | Mixed OS, untrusted workloads, kernels |

**Rule of thumb:** use a container when the workload is Linux, trusted, and
needs to be lean. Use a VM when it needs its own kernel, a non-Linux OS, or
stronger isolation.

---

## 3. Container templates

### Concept

A container template is a compressed root filesystem tarball (`.tar.zst`,
`.tar.gz`, `.tar.xz`) containing a minimal OS userland: no kernel, no boot
loader. Proxmox downloads them with **`pveam`** (Proxmox VE Appliance
Manager) into a storage that has the `vztmpl` content type enabled.

Default location on a standard install: `/var/lib/vz/template/cache/`.

### Practical implementation

```bash
# Refresh the template catalogue (network access required)
pveam update

# Browse the catalogue
pveam available --section system
pveam available --section turnkeylinux

# Download into storage "local" (example name — confirm with pveam available)
pveam download local debian-13-standard_13.1-1_amd64.tar.zst

# List what is actually cached locally
pveam list local

# Remove a cached template
pveam remove local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst
```

### Evidence and troubleshooting

- Record the `pveam available` and `pveam list local` output in the lab notes.
- `no such template` → run `pveam update`, then download again.
- `storage does not support content type vztmpl` → enable `vztmpl` on that
  storage under **Datacenter → Storage**.
- Download failures are usually DNS, proxy, or egress firewall issues on the
  host — test with `curl -I https://download.proxmox.com`.

---

## 4. Container creation

### Concept

`pct create <CTID> <template> [options]` allocates a root filesystem volume,
writes `/etc/pve/lxc/<CTID>.conf`, and (optionally) starts the container.
Every option is also available later through `pct set`.

### Key parameters

| Parameter | Meaning |
|---|---|
| `CTID` | Unique integer ID. Not shared with VM IDs; 100–999 is conventional |
| `--hostname` | Container hostname, must be a valid DNS label |
| `--cores` | Number of visible CPUs |
| `--memory` | RAM in MiB |
| `--swap` | Swap in MiB (`0` disables swap) |
| `--rootfs` | Root filesystem, e.g. `local-lvm:8` (8 GiB thin volume) |
| `--net0` | Network interface definition |
| `--unprivileged` | `1` for user-namespace isolation (recommended) |
| `--features` | `nesting=1,keyctl=1,fuse=1,mknod=1` |
| `--onboot` | Start automatically with the node |
| `--start` | Start immediately after creation |
| `--password` / SSH keys | Initial access method |

### Practical implementation

```bash
pct create 200 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-200 \
  --cores 2 --memory 1024 --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=192.0.2.200/24,gw=192.0.2.1 \
  --unprivileged 1 \
  --features nesting=1 \
  --onboot 0 \
  --start 1
```

### Evidence and troubleshooting

- Capture `pct config 200` immediately after creation — it is the single most
  useful troubleshooting artefact.
- `CTID already exists` → choose another ID; check `pct list`.
- `unable to create volume` → the target storage is full or does not support
  `rootdir`.
- Creation succeeds but start fails → see §12.

---

## 5. Privileged vs unprivileged containers

### Concept

This is the most important security decision for a container.

- **Privileged:** UID 0 inside the container **is** UID 0 on the host. A
  container root can, in principle, reach host resources. Simpler, but risky.
- **Unprivileged:** the container runs inside a **user namespace**. UID 0 in
  the container maps to an unprivileged host UID (typically `100000`), so
  container root is *not* host root.

```text
Unprivileged default mapping (/etc/pve/lxc/<CTID>.conf):
lxc.idmap: u 0 100000 65536    # container UID 0..65535 -> host 100000..165535
lxc.idmap: g 0 100000 65536
```

### Practical implementation

```bash
# New container, unprivileged (preferred)
pct create 202 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-202 --unprivileged 1 --rootfs local-lvm:8 --start 1

# Inspect the mapping
pct config 202
grep -E '^lxc.idmap' /etc/pve/lxc/202.conf

# Host-side identity check: container root -> host UID 100000
pct exec 202 -- id
ps -o user,uid,cmd -C sleep,nginx | head
```

> **Note:** `pct create` defaults to **privileged** unless you pass
> `--unprivileged 1`. The web UI defaults to unprivileged — a mismatch that
> catches people out when mixing GUI and CLI workflows.

### Evidence and troubleshooting

- Record `pct config` for both containers and the `id` output from inside each.
- Converting an existing container's privilege mode is **not supported**.
  Recreate and migrate data instead.
- Unprivileged containers cannot use some host devices or write to host paths
  owned by un-mapped UIDs. That is the trade-off, not a bug.
- For bind mounts that need specific host ownership, extend `/etc/subuid` and
  `/etc/subgid` on the host and add an extra `lxc.idmap` line.

---

## 6. Container networking

### Concept

Proxmox creates a `veth` pair per container interface: one end inside the
container (`eth0`), the other attached to a Linux bridge on the host
(`vmbr0`, `vmbr1`). The bridge behaves as a virtual switch.

| Mode | `--net0` value | Use |
|---|---|---|
| DHCP | `name=eth0,bridge=vmbr0,ip=dhcp` | Quick lab access |
| Static | `name=eth0,bridge=vmbr0,ip=192.0.2.205/24,gw=192.0.2.1` | Predictable addressing |
| VLAN tag | `...,tag=10` | Segment on a VLAN-aware bridge |
| No network | `name=eth0,bridge=vmbr0,link_down=1` or omit | Isolated testing |
| Rate limit | `...,rate=10` | Cap at 10 MB/s |
| Firewall | `...,firewall=1` | Apply Proxmox firewall rules |

VLAN tagging **requires** the bridge to be VLAN-aware (`bridge-vlan-aware yes`
in `/etc/network/interfaces`), which is already the case for `vmbr1` in this
lab.

### Practical implementation

```bash
# Static address on the management bridge
pct set 203 --net0 name=eth0,bridge=vmbr0,ip=192.0.2.203/24,gw=192.0.2.1

# VLAN 10 on the isolated VLAN-aware bridge
pct set 203 --net0 name=eth0,bridge=vmbr1,tag=10,ip=198.51.100.10/24

# DHCP
pct set 203 --net0 name=eth0,bridge=vmbr0,ip=dhcp

# Verify from the host
pct exec 203 -- ip -4 addr show eth0
pct exec 203 -- ip route
pct exec 203 -- ping -c 3 192.0.2.1
bridge link show | grep -E 'vmbr0|vmbr1'
```

### Evidence and troubleshooting

- Capture: `pct config`, `ip a`, `ip route`, `bridge link show`, and a ping/
  traceroute result. This is exactly the network evidence list in
  [`labs/README.md`](../labs/README.md).
- No address → check the bridge has a carrier, the tag matches an allowed
  VLAN, and `link_down` is not set.
- DHCP timeout → no DHCP server reachable on that segment; try a static IP to
  isolate the cause.
- Changing `--net0` while the container runs applies live for most drivers,
  but a `pct reboot` makes the change deterministic.

---

## 7. Container storage

### Concept

A container has exactly **one root filesystem** (`rootfs`) plus up to ten
optional **mount points** (`mp0`–`mp9`). Unlike a VM there is no separate
virtual disk controller, no boot order, no ISO.

| Type | Example | Notes |
|---|---|---|
| Thin volume | `local-lvm:8` | Snapshot-capable; grows on demand |
| Directory | `local:8` | File on a directory storage; no snapshots on plain dir |
| Bind mount | `mp0 /srv/data,mp=/data` | Host path mapped into the container |
| Extra volume | `mp0 local-lvm:4,mp=/data` | Separate volume; survives rootfs rebuild |

Second disk-like mount points are how you keep data independent of the
container root filesystem — important for backup and restore strategy.

### Practical implementation

```bash
# Inspect storage and its content types
pvesm status
pvesm list local | head
cat /etc/pve/storage.cfg

# Grow the root filesystem (grow only, never shrink)
pct resize 204 rootfs +2G
pct df 204

# Add a 4 GiB data volume at /data
pct set 204 --mp0 local-lvm:4,mp=/data

# Bind mount a host directory (read-only example)
pct set 204 --mp1 /srv/lab-share,mp=/mnt/share,ro=1

# Trim free blocks on thin storage
pct fstrim 204
```

### Evidence and troubleshooting

- Capture `pct config 204`, `pct df 204`, `pvesm status`, and `df -h` inside.
- `no space left on device` while `pct df` looks fine → the underlying thin
  pool is full; check `lvs` and `pvesm status` on the host.
- Bind mounts fail for unprivileged containers when the host path is owned by
  an unmapped UID — fix ownership or extend the ID map.
- Snapshots require thin/ZFS-backed storage; plain directory storage cannot
  snapshot.

---

## 8. Container lifecycle

### Concept

Container states are `running`, `stopped`, and transient `locked`. Operations
map onto the guest's init system: `stop` is effectively a kill, `shutdown`
sends ACPI-like signals through `lxc-attach` and waits.

### Practical implementation

```bash
pct list                                  # all containers + status
pct status 200                            # running / stopped
pct start 200
pct shutdown 200 --timeout 60             # graceful
pct stop 200                              # immediate
pct reboot 200                            # graceful restart
pct destroy 200 --purge                   # delete config + disks (irreversible)
pct unlock 200                            # clear a stale lock
```

### Evidence and troubleshooting

- Record `pct list` and `pct status` output before and after each operation.
- `shutdown` hangs → the guest init is not responding; `pct stop` is the
  fallback. Note the timeout value you used.
- `CT is locked (backup|snapshot|migrate)` → wait for the task to finish or
  `pct unlock` after confirming no task is running (`/var/log/pve/tasks/`).
- `pct destroy` is permanent. Snapshot or back up first.

---

## 9. Container backup and restore

### Concept

`vzdump` archives the container's configuration and root filesystem into a
single compressed file on a storage with the `backup` content type. Restore
is `pct restore` (containers) — `qmrestore` is for VMs only.

Two snapshot-internal modes matter:

- `--mode snapshot` — no downtime, uses storage snapshots when available.
- `--mode stop` — container is shut down and restarted around the backup.

### Practical implementation

```bash
# One-off backup
vzdump 200 --storage local --mode snapshot --compress zstd

# Restore as a new CTID, allowing a different rootfs size/storage
pct restore 208 local:backup/vzdump-lxc-200-<timestamp>.tar.zst \
  --storage local-lvm --rootfs local-lvm:8 --unprivileged 1

# Scheduled job retention (keep last 7 daily, 4 weekly)
# Datacenter -> Backup -> Add, or:
cat /etc/pve/jobs.cfg
```

### Evidence and troubleshooting

- Capture: the `vzdump` task log summary, resulting file listing with sizes,
  and the `pct config` of the restored container.
- Restore failing on `--rootfs` size → the target volume must be at least as
  large as the archived filesystem.
- A backup is not verified until it has been restored. Always restore to a new
  CTID and confirm service state before trusting it.
- Retention is enforced by the job, not by `vzdump` on its own — document the
  policy you chose.

---

## 10. Resource management

### Concept

Containers expose three main knobs:

| Knob | Option | Behaviour |
|---|---|---|
| CPU count | `--cores` | Number of CPUs visible to the guest |
| CPU cap | `--cpulimit` | Hard ceiling in cores; `0` = unlimited |
| CPU weight | `--cpuunits` | Relative share under contention (default 1024) |
| Memory | `--memory` | Hard RAM limit in MiB |
| Swap | `--swap` | Swap allowance; `0` disables swap entirely |

`--cores` does not reserve CPU. `--cpulimit` is the real cap. Memory is a hard
limit in cgroup v2, which is what Proxmox VE 9 uses exclusively.

### Practical implementation

```bash
pct set 205 --cores 2 --cpulimit 1.5 --cpuunits 512
pct set 205 --memory 1024 --swap 512

# Observe limits from inside
pct exec 205 -- nproc
pct exec 205 -- cat /sys/fs/cgroup/memory.max
pct exec 205 -- free -m

# Stress test (if installed in the container) to prove the cap
pct exec 205 -- apt-get install -y stress-ng
pct exec 205 -- stress-ng --cpu 4 --timeout 60s
```

### Evidence and troubleshooting

- Capture before/after `free -m`, `nproc`, and the stress test result — "the
  limit exists" is unproven until you exceed it.
- OOM kills inside the container are expected when `--memory` is too low for
  the workload; check `dmesg | grep -i oom`.
- `--swap 0` makes memory exhaustion fail fast instead of swapping on the host
  disk. Good for predictable behaviour, bad for bursty workloads.

---

## 11. Container security

### Concept

Defence in depth, in order of impact:

1. **Unprivileged by default** — the biggest single win.
2. **AppArmor** — Proxmox generates a per-container profile; blocks unsafe
   syscalls and host path access.
3. **No `nesting` unless required** — each enabled feature widens the attack
   surface.
4. **Firewall + network segmentation** — `firewall=1` on the NIC, VLAN or
   isolated bridge for untrusted services.
5. **Least privilege inside** — avoid root logins, prefer SSH keys, minimal
   services.
6. **Host-side controls** — RBAC, 2FA, API tokens scoped to a role, no shared
   root password.

### Practical implementation

```bash
# Show isolation-relevant settings
pct config 207 | grep -E 'unprivileged|features|net0'
cat /sys/module/apparmor/parameters/enabled
aa-status | grep -i lxc
dmesg | grep -i apparmor | tail

# Read-only bind mount instead of read-write
pct set 207 --mp0 /srv/lab-share,mp=/mnt/share,ro=1

# Firewall on the container NIC
pct set 207 --net0 name=eth0,bridge=vmbr1,tag=20,ip=203.0.113.20/24,firewall=1
```

### Evidence and troubleshooting

- Capture `pct config`, AppArmor status, and the firewall rule set in use.
- `Permission denied` from a service → AppArmor denial appears in `dmesg`;
  do not disable AppArmor globally, adjust the feature flags or the profile.
- Enabling `nesting=1` for Docker also relaxes AppArmor in ways you should
  document explicitly.
- Never publish unredacted screenshots of the Proxmox login, tokens, or
  container shells showing credentials.

---

## 12. Troubleshooting

A structured approach beats guessing. Work outward: config → storage →
network → AppArmor → guest init.

### First five commands for any container problem

```bash
pct status <CTID>
pct config <CTID>
pct df <CTID>
journalctl -u pve-container@<CTID> --no-pager -n 50
dmesg | tail -50
```

### Symptom → likely cause

| Symptom | Check first |
|---|---|
| `pct start` fails immediately | `journalctl -u pve-container@<CTID>`, AppArmor denials in `dmesg` |
| `CT is locked` | Task still running in `/var/log/pve/tasks/`; then `pct unlock` |
| No network inside | `pct config` net line, `bridge link show`, bridge carrier, VLAN tag |
| DHCP never resolves | DHCP server reachable on that segment? test static |
| Write failures / disk full | `pct df`, `pvesm status`, `lvs` on host |
| Service works then dies | cgroup memory limit (`memory.max`), OOM in `dmesg` |
| Permission denied on bind mount | UID mapping for unprivileged containers |
| Container cannot reach internet but host can | `ip_forward`, NAT/`masquerade`, container gateway |

### Evidence discipline

For every troubleshooting episode record: the symptom, the exact commands run,
the output that identified the cause, the fix applied, and the verification
that proved it fixed. That sequence is the most valuable content in the whole
repository.

---

## Command reference (`pct`)

| Task | Command |
|---|---|
| List containers | `pct list` |
| Status | `pct status <id>` |
| Show config | `pct config <id>` |
| Change config | `pct set <id> --memory 2048` |
| Start / stop / shutdown / reboot | `pct start|stop|shutdown|reboot <id>` |
| Destroy | `pct destroy <id> --purge` |
| Clear lock | `pct unlock <id>` |
| Shell inside | `pct enter <id>` |
| Run one command | `pct exec <id> -- <cmd>` |
| Copy file in / out | `pct push <id> <local> <remote>` / `pct pull <id> <remote> <local>` |
| Resize rootfs | `pct resize <id> rootfs +2G` |
| Disk usage | `pct df <id>` |
| Snapshot | `pct snapshot <id> <name>` |
| Rollback | `pct rollback <id> <name>` |
| List / delete snapshot | `pct listsnapshot <id>` / `pct delsnapshot <id> <name>` |
| Backup (all content) | `vzdump <id> --storage <store> --mode snapshot` |
| Restore | `pct restore <newid> <archive> --storage <store>` |
| Migrate (cluster) | `pct migrate <id> <target-node>` |
| Template operations | `pveam update|available|download|list|remove` |

---

## Status in this lab

| Topic | Where it is tracked |
|---|---|
| Concepts 1–12 | this document |
| Practical execution | [`labs/lxc/`](../labs/lxc/) — status column in the lab index |
| Evidence | [`evidence/lxc/`](../evidence/lxc/) |
| Advanced follow-on work | [`advanced-topics.md`](advanced-topics.md) |
| Wider Proxmox roadmap | [`roadmap.md`](roadmap.md) |

Nothing in this document claims a lab was executed. Configuration notes about
`vmbr0`, `vmbr1` and VM 100 live in
[`proxmox-learning-notes.md`](proxmox-learning-notes.md).
