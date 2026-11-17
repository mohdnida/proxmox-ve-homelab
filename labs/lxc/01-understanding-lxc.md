# Lab 01 — Understanding LXC

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | none (inspection only) |
| Storage used | read-only inspection |
| Network | read-only inspection |
| Prerequisites | Working shell on the Proxmox node |
| Estimated time | 30–45 min |
| Evidence folder | `evidence/lxc/01-understanding-lxc/` |

## Concept

A container is a set of Linux processes confined by **namespaces** (what they
can see) and **cgroups** (what they can use). No second kernel is booted, so
the container's `uname -r` reports the host kernel, exactly like the host.

The purpose of this lab is to prove that statement on this node instead of
assuming it, and to establish the baseline commands every later lab depends on.

## Prerequisites

- [ ] Proxmox node reachable at `192.0.2.100/24`, shell access as `root`.
- [ ] `pct` available (it ships with Proxmox VE).
- [ ] No containers exist yet — this lab creates none.

## Practical implementation

### 1. Confirm the tooling

```bash
pct --version
pct help
lxc-ls --version
uname -r
```

### 2. Inspect the kernel features containers rely on

```bash
# Namespace types the running kernel supports
ls /proc/self/ns
lsns | head -20

# cgroups: Proxmox VE 9 uses cgroup v2 only
mount | grep cgroup
ls /sys/fs/cgroup | head -20
stat -fc %T /sys/fs/cgroup        # expect: cgroup2fs
cat /proc/self/cgroup
```

### 3. Inspect the container configuration surface

```bash
ls -la /etc/pve/lxc/              # empty before Lab 03
cat /etc/pve/storage.cfg
pvesm status
pct list
```

### 4. Stand up the comparison with an existing VM

VM 100 exists from earlier work. Compare what "virtual machine" means here:

```bash
qm list
qm config 100
qm status 100
```

### 5. Read the filesystem layout

```bash
df -h /var/lib/vz
ls -la /var/lib/vz/template/cache/ 2>/dev/null    # templates: empty before Lab 02
ls -la /var/lib/vz/dump/ 2>/dev/null              # backups: empty before Lab 08
```

### 6. Write your own definition

In `evidence/lxc/01-understanding-lxc/05-verification.md`, answer in two or
three sentences each:

1. What does a container share with the host, and what is isolated?
2. Which command proves a container has no kernel of its own?
3. Why can this node not run a Windows container?
4. What are the two subsystems that make LXC possible?

## Verification

- [ ] `stat -fc %T /sys/fs/cgroup` returns `cgroup2fs`.
- [ ] `ls /etc/pve/lxc/` is empty, and `pct list` shows no containers.
- [ ] `qm list` shows VM 100, demonstrating VM and CT are separate ID spaces.
- [ ] You can name the namespace (`user`) that makes an unprivileged container
      safe, before Lab 04 demonstrates it.

## Evidence to capture

- [ ] `01-config.txt` — `pct --version`, `lxc-ls --version`, `uname -r`
- [ ] `02-commands.md` — commands run, in order
- [ ] `03-output.txt` — `lsns`, cgroup checks, `pvesm status`, `qm config 100`
- [ ] `04-network.txt` — `ip -br a` (baseline before any container exists)
- [ ] `05-verification.md` — your answers to the six questions
- [ ] `06-troubleshooting.md` — anything unexpected

## Troubleshooting log

| Symptom | Commands used | Root cause | Fix | Verified |
|---|---|---|---|---|
| `lsns` not found | `apt list --installed \| grep util-linux` | missing package | `apt install util-linux` | |
| `/etc/pve/lxc/` missing | `ls /etc/pve` | `pve-cluster` service down | `systemctl status pve-cluster` | |

## Key takeaways

- Containers are processes, not simulated machines.
- The host kernel version is the container kernel version — always.
- Baseline observations before creating anything are evidence, not busywork.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §1–2
- Next: [`02-downloading-templates.md`](02-downloading-templates.md)
