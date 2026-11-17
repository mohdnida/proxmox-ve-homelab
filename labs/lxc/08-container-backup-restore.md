# Lab 08 — Container Backup and Restore

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | CT 206 (source), CT 208 (restore target, temporary) |
| Storage used | `local` (`backup` content), `local-lvm` (rootfs) |
| Network | `vmbr0`, `192.0.2.206/24` |
| Prerequisites | Lab 03; CT 204's `/data` volume is useful for a partial-restore test |
| Estimated time | 45–60 min |
| Evidence folder | `evidence/lxc/08-container-backup-restore/` |

## Concept

`vzdump` archives a container's configuration **and** root filesystem into one
compressed file on a storage with the `backup` content type. Restore is
`pct restore` — `qmrestore` is only for VMs.

The critical idea: **a backup you have not restored is not a backup.** This lab
ends with a verified restore to a new CT ID, not with a finished `vzdump` task.

## Prerequisites

- [ ] `local` has `backup` in its content types.
- [ ] Free space: one archive is roughly the size of the used data.
- [ ] `192.0.2.206` free; CT 208 not in use.

## Practical implementation

### 1. Create a container with something worth losing

```bash
pct create 206 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-206-backup --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=192.0.2.206/24,gw=192.0.2.1 \
  --unprivileged 1 --start 1

pct exec 206 -- bash -c 'echo "state before backup" > /root/proof.txt; \
  date -u > /root/timestamp.txt; mkdir -p /srv/app; echo v1 > /srv/app/app.conf'
pct exec 206 -- cat /root/proof.txt
```

### 2. Take a snapshot-mode backup

```bash
vzdump 206 --storage local --mode snapshot --compress zstd
```

Note the task ID and watch it complete (`/var/log/pve/tasks/` or the web UI).
Then confirm the artefact:

```bash
ls -lh /var/lib/vz/dump/ | grep 206
```

### 3. Change the container so the restore is meaningful

```bash
pct exec 206 -- bash -c 'echo "state AFTER backup" > /root/proof.txt; \
  rm -f /srv/app/app.conf; echo broken > /srv/app/broken.txt'
pct exec 206 -- cat /root/proof.txt
```

### 4. Restore to a new CT ID

```bash
# Use the exact archive name produced by vzdump
pct restore 208 local:backup/vzdump-lxc-206-<TIMESTAMP>.tar.zst \
  --storage local-lvm --rootfs local-lvm:4 --unprivileged 1 --hostname lab-ct-208-restored

pct start 208
pct config 208
pct exec 208 -- cat /root/proof.txt
pct exec 208 -- ls -l /srv/app/
```

Expected: `state before backup` is back, `app.conf` exists, `broken.txt` does
not. That is a verified restore.

### 5. Prove the restored data is a real, independent copy

```bash
pct exec 208 -- bash -c 'echo "only in restored copy" > /root/only-208.txt'
pct exec 206 -- ls /root/only-208.txt    # must not exist
```

### 6. Configure retention properly

```bash
# Scheduled job with retention: keep 7 daily, 4 weekly
pve-backup-job-add 2>/dev/null || echo "configure the job in the web UI"
cat /etc/pve/jobs.cfg 2>/dev/null
```

If the CLI helper is unavailable, create the job under
**Datacenter → Backup → Add** with `vzdump` mode `snapshot`, storage `local`,
and retention `keep-daily=7, keep-weekly=4`. Then record the resulting
`/etc/pve/jobs.cfg` stanza as evidence.

### 7. Clean up the temporary restore

```bash
pct stop 208
pct destroy 208 --purge
pct list
```

Keep the archive from step 2 until you have finished Lab 10.

## Verification

- [ ] `vzdump` task completed successfully and the archive exists.
- [ ] The restored container shows the *pre-change* state.
- [ ] A file created in the restore does not exist in the source.
- [ ] The retention policy is visible in the job configuration.
- [ ] Temporary CT 208 destroyed, source CT 206 intact.

## Evidence to capture

- [ ] `01-config.txt` — `pct config 206` and `pct config 208`
- [ ] `02-commands.md` — `vzdump` and `pct restore` exactly as run
- [ ] `03-output.txt` — task summary, archive listing, `proof.txt` contents
- [ ] `05-verification.md` — a table of expected vs observed state
- [ ] `06-troubleshooting.md` — any restore failure and its cause
- [ ] Screenshots — backup task log and restored container, redacted

## Troubleshooting log

| Symptom | Commands used | Root cause | Fix | Verified |
|---|---|---|---|---|
| `storage does not support content type backup` | `cat /etc/pve/storage.cfg` | content type missing | enable `backup` | |
| Restore fails on volume size | `pct restore ... --rootfs` | target smaller than archived data | allocate at least as large | |
| `mode snapshot` falls back to stop | task log | storage cannot snapshot | expected on dir storage; document it | |
| Restore starts but network is wrong | `pct config 208` | new CT ID/addresses not set | restore then `pct set --net0` | |
| Archive name not found | `ls /var/lib/vz/dump/` | wrong timestamp | copy the exact filename | |

## Key takeaways

- `vzdump` produces a container archive; `pct restore` consumes it.
- Restore validates; a completed backup task alone proves nothing.
- Retention belongs to the job, not to a manual `vzdump`.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §9
- [`proxmox-learning-notes.md`](../../docs/proxmox-learning-notes.md) §16
- Next: [`09-container-security.md`](09-container-security.md)
