# Lab 06 — Container Storage

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | CT 204 |
| Storage used | `local-lvm` rootfs 4 GiB, `local-lvm` 4 GiB mount point, host bind mount |
| Network | `vmbr0`, `192.0.2.204/24` |
| Prerequisites | Lab 03 |
| Estimated time | 45–60 min |
| Evidence folder | `evidence/lxc/06-container-storage/` |

## Concept

A container has exactly one **root filesystem** and up to ten **mount points**
(`mp0`–`mp9`). There is no disk controller, no boot device, no ISO — a
container's "disks" are paths in its mount namespace.

Keeping data on a separate mount point is what makes a container disposable:
you can destroy the root filesystem and reattach the data volume.

## Prerequisites

- [ ] Thin-pool free space for 4 GiB rootfs + 4 GiB volume.
- [ ] `192.0.2.204` free.
- [ ] `pvesm status` shows `local-lvm` active.

## Practical implementation

### 1. Baseline storage

```bash
pvesm status
cat /etc/pve/storage.cfg
lvs
df -h /var/lib/vz
```

Note the content types per storage: `local` has `vztmpl`, `backup`, `iso`;
`local-lvm` has `rootdir`, `images`.

### 2. Create the container

```bash
pct create 204 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-204-storage --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=192.0.2.204/24,gw=192.0.2.1 \
  --unprivileged 1 --start 1

pct df 204
pct exec 204 -- df -hT /
```

### 3. Grow the root filesystem

Growing is supported; shrinking is not.

```bash
pct resize 204 rootfs +2G
pct df 204
pct exec 204 -- df -hT /

# Proof that shrink is refused
pct resize 204 rootfs -1G
```

Record the exact error message from the last command — it is useful evidence
that the constraint is real rather than a rumour.

### 4. Add a separate data volume

```bash
pct set 204 --mp0 local-lvm:4,mp=/data
pct reboot 204
pct config 204 | grep mp0
pct exec 204 -- findmnt /data
pct exec 204 -- df -hT /data

# Put something in it that must survive a rootfs rebuild
pct exec 204 -- bash -c 'echo kept > /data/keep-me.txt'
pct exec 204 -- cat /data/keep-me.txt
```

### 5. Add a read-only bind mount from the host

```bash
mkdir -p /srv/lab-share
echo "shared host content" > /srv/lab-share/hello.txt
chmod 755 /srv/lab-share

pct set 204 --mp1 /srv/lab-share,mp=/mnt/share,ro=1
pct reboot 204

pct exec 204 -- cat /mnt/share/hello.txt
pct exec 204 -- touch /mnt/share/should-fail    # must fail: read-only
pct exec 204 -- findmnt -T /mnt/share
```

### 6. Observe thin provisioning honestly

```bash
lvs -o lv_name,lv_size,data_percent,metadata_percent | grep -E 'vm-204|data'
pct fstrim 204
pct df 204
```

`pct df` reports the filesystem inside the container; `lvs` reports actual
thin-pool consumption. Learn to read both, because "disk full" symptoms
usually come from the pool, not the filesystem.

### 7. Optional: move the rootfs to directory storage

```bash
# Moves the container's rootfs to a directory storage — slower, no snapshots
pct move_volume 204 rootfs local:4
pct config 204 | grep rootfs
pvesm list local | grep 204

# Move it back to thin storage
pct move_volume 204 rootfs local-lvm:4
```

If the move is not supported in your configuration, record the error instead —
that is equally valid evidence.

## Verification

- [ ] `pct df 204` shows the +2 GiB growth.
- [ ] `pct resize ... -1G` fails with a clear message.
- [ ] `/data` is a distinct filesystem (`findmnt`) and `keep-me.txt` persists.
- [ ] A write to `/mnt/share` fails with `Read-only file system`.
- [ ] You can explain the difference between `pct df` and `lvs` output.

## Evidence to capture

- [ ] `01-config.txt` — `pct config 204` after each change
- [ ] `02-commands.md` — commands in order
- [ ] `03-output.txt` — `df -hT`, `findmnt`, resize errors, `lvs`
- [ ] `05-verification.md` — the five checks with results
- [ ] `06-troubleshooting.md` — problems and fixes
- [ ] Screenshots — Proxmox storage view and container `df`, redacted

## Troubleshooting log

| Symptom | Commands used | Root cause | Fix | Verified |
|---|---|---|---|---|
| `pct resize` fails on shrink | — | shrink unsupported | recreate smaller or restore | |
| Write to bind mount denied | `findmnt`, `pct config` | unprivileged UID mapping | fix host ownership / idmap | |
| `no space left` with free-looking `pct df` | `lvs`, `pvesm status` | thin pool full | free space, remove snapshots, resize pool | |
| `mp0` not visible | `pct config`, `pct status` | change requires restart | `pct reboot 204` | |
| `move_volume` refused | `pvesm status` | target storage lacks `rootdir` | enable content type | |

## Key takeaways

- Separate data from the root filesystem from day one.
- Resize is grow-only; plan initial sizes accordingly.
- Thin-pool exhaustion is a host-wide failure, not a container-level one.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §7
- Next: [`07-resource-management.md`](07-resource-management.md)
