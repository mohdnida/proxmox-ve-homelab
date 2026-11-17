# Lab 03 — Creating Your First Container

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | CT 200 |
| Storage used | `local-lvm` (rootfs), `local` (template) |
| Network | `vmbr0`, static `192.0.2.200/24`, gw `192.0.2.1` |
| Prerequisites | Lab 02, one template cached |
| Estimated time | 30–45 min |
| Evidence folder | `evidence/lxc/03-creating-first-container/` |

## Concept

`pct create` allocates a thin root filesystem volume, writes
`/etc/pve/lxc/<CTID>.conf`, and optionally starts the container. Unlike a VM
there is no disk controller, no boot order and no ISO — just a filesystem and
a config file.

This lab deliberately uses **unprivileged** mode from the start, because the
default for `pct create` is privileged and that default is a trap.

## Prerequisites

- [ ] Template name confirmed in Lab 02 (`pveam list local`).
- [ ] Space on `local-lvm` for an 8 GiB thin volume.
- [ ] `192.0.2.200` is free on the management segment.
- [ ] CT 200 does not already exist (`pct list`).

## Practical implementation

### 1. Record the pre-state

```bash
pct list
pvesm status
free -m
```

### 2. Create the container

```bash
pct create 200 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-200 \
  --cores 2 --memory 1024 --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=192.0.2.200/24,gw=192.0.2.1 \
  --unprivileged 1 \
  --features nesting=1 \
  --onboot 0 \
  --password
```

Enter a temporary password when prompted. Replace it or add an SSH key before
this container ever becomes long-lived.

### 3. Read the generated configuration

```bash
pct config 200
cat /etc/pve/lxc/200.conf
pct status 200
```

Notice: `unprivileged: 1`, `rootfs: local-lvm:vm-200-disk-0,size=8G`, and the
`net0` line. These three lines are what later labs modify.

### 4. Start it and get a shell

```bash
pct start 200
pct status 200
pct exec 200 -- id
pct exec 200 -- uname -r
pct enter 200
```

Inside the container, note that `uname -r` matches the **host** kernel. That is
the proof from Lab 01, now observed live.

```bash
# inside the container
ip -4 addr show eth0
ip route
cat /etc/hostname
exit
```

### 5. Prove it is a real Debian userland

```bash
pct exec 200 -- cat /etc/os-release
pct exec 200 -- apt-get update
pct exec 200 -- apt-get install -y curl
pct exec 200 -- curl -sI https://download.proxmox.com | head -3
```

### 6. Resource footprint, host side

```bash
free -m                     # 1024 MiB reserved for the container
pct df 200
lvs | grep vm-200
```

### 7. Clean shutdown

```bash
pct shutdown 200 --timeout 60
pct status 200
```

Leave CT 200 **stopped** for Lab 04, or start it again if you prefer — record
the state you left it in.

## Verification

- [ ] `pct config 200` shows `unprivileged: 1`.
- [ ] `pct exec 200 -- uname -r` equals `uname -r` on the host.
- [ ] `pct exec 200 -- ip -4 addr` shows `192.0.2.200/24`.
- [ ] Outbound HTTPS from inside the container works (curl headers returned).
- [ ] `pct shutdown` returns the container to `stopped` cleanly.

## Evidence to capture

- [ ] `01-config.txt` — `pct config 200`, `/etc/pve/lxc/200.conf`
- [ ] `02-commands.md` — the create command exactly as run
- [ ] `03-output.txt` — `uname -r` host vs container, `id`, os-release
- [ ] `04-network.txt` — `ip -4 a`, `ip route`, host `bridge link`
- [ ] `05-verification.md` — the five checklist results
- [ ] `06-troubleshooting.md` — any failures
- [ ] Screenshots — container summary and console, **redacted**

## Troubleshooting log

| Symptom | Commands used | Root cause | Fix | Verified |
|---|---|---|---|---|
| `CT 200 already exists` | `pct list` | ID in use | use another ID or destroy the stale one | |
| `unable to create volume` | `pvesm status`, `lvs` | thin pool full | free space, resize, or pick another storage | |
| `pct exec` hangs | `pct status 200`, `journalctl -u pve-container@200` | container not running | `pct start 200` | |
| No network inside | `pct config 200`, `bridge link show` | wrong bridge/gateway or `link_down=1` | correct `--net0` | |
| Start fails, config parse error | `journalctl -u pve-container@200 -n 30` | malformed `/etc/pve/lxc/200.conf` | fix the line, retry | |

## Key takeaways

- `pct create` defaults to privileged — always pass `--unprivileged 1`.
- The container config file is the whole truth; everything else is derived.
- An unprivileged container still looks like root inside. That is the point of
  Lab 04.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §4
- Next: [`04-privileged-vs-unprivileged.md`](04-privileged-vs-unprivileged.md)
