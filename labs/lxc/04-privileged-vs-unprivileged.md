# Lab 04 — Privileged vs Unprivileged Containers

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | CT 201 (privileged), CT 202 (unprivileged) |
| Storage used | `local-lvm` 4 GiB each |
| Network | `vmbr0`, `192.0.2.201/24` and `192.0.2.202/24` |
| Prerequisites | Lab 03 |
| Estimated time | 45–60 min |
| Evidence folder | `evidence/lxc/04-privileged-vs-unprivileged/` |

## Concept

Privileged and unprivileged containers look identical from inside. The
difference is **UID mapping**:

- Privileged: container UID 0 → host UID 0. Container root **is** host root.
- Unprivileged: container UID 0 → host UID 100000, via a user namespace. The
  container root is an ordinary unprivileged user on the host.

The purpose of this lab is to observe that difference rather than read about
it, and to see exactly which host resources each mode can touch.

> **Safety:** this lab demonstrates a privilege difference on purpose. Only do
> the privileged write test on a disposable lab node, never on a host you care
> about, and delete the test path afterwards.

## Prerequisites

- [ ] Lab 03 complete, CT 200 exists.
- [ ] Free thin-pool space for two 4 GiB containers.
- [ ] `192.0.2.201` and `192.0.2.202` free.

## Practical implementation

### 1. Create both containers

```bash
# Privileged (note: --unprivileged 1 is absent, this is the point)
pct create 201 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-201-priv --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=192.0.2.201/24,gw=192.0.2.1 \
  --password

# Unprivileged
pct create 202 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-202-unpriv --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=192.0.2.202/24,gw=192.0.2.1 \
  --unprivileged 1 \
  --password
```

### 2. Compare the configuration

```bash
pct config 201 | grep -E 'unprivileged|idmap|rootfs'
pct config 202 | grep -E 'unprivileged|idmap|rootfs'
cat /etc/pve/lxc/202.conf
```

Expected: CT 202 has `unprivileged: 1` and two generated `lxc.idmap` lines;
CT 201 has neither.

### 3. Observe the UID mapping from both sides

```bash
pct start 201; pct start 202

# Inside each container, root looks the same
pct exec 201 -- id
pct exec 202 -- id

# On the host, start a long sleep in each and inspect the owning UID
pct exec 201 -- systemd-run --unit=probe201 sleep 300
pct exec 202 -- systemd-run --unit=probe202 sleep 300

ps -o pid,user,uid,cmd -C sleep
```

Expected: the processes for CT 202 are owned by host UID **100000**, while
CT 201's are owned by **root**.

Note the PID reported for each — it is the host PID, because the container's
processes are host processes.

### 4. Host filesystem access test

Create a host-only path that is root-owned and mode `700`:

```bash
mkdir -p /srv/lab-root-only
chmod 700 /srv/lab-root-only
chown root:root /srv/lab-root-only
echo "host secret" > /srv/lab-root-only/host-file
```

Mount it into both containers:

```bash
pct set 201 --mp0 /srv/lab-root-only,mp=/mnt/hostroot
pct set 202 --mp0 /srv/lab-root-only,mp=/mnt/hostroot
pct restart 201; pct restart 202
```

```bash
# Privileged container: reads the host root-only file
pct exec 201 -- cat /mnt/hostroot/host-file
pct exec 201 -- touch /mnt/hostroot/written-by-201

# Unprivileged container: permission denied
pct exec 202 -- cat /mnt/hostroot/host-file
pct exec 202 -- touch /mnt/hostroot/written-by-202
```

Expected: CT 201 succeeds; CT 202 gets `Permission denied`. This is the
security difference, observed.

### 5. Clean up the dangerous demonstration

```bash
pct exec 201 -- rm -f /mnt/hostroot/written-by-201
pct set 201 --delete mp0
pct set 202 --delete mp0
rm -rf /srv/lab-root-only
pct restart 201; pct restart 202
```

### 6. Note the supported/unsupported facts

- Converting an existing container between privileged and unprivileged is **not
  supported**. You recreate and migrate data.
- The web UI defaults to unprivileged; `pct create` defaults to privileged.
  Confirm in `pct config` rather than assuming.
- For bind mounts needing specific ownership, add `lxc.idmap` entries and
  matching lines in `/etc/subuid` and `/etc/subgid` on the host.

## Verification

- [ ] `ps -o uid -C sleep` shows host UID 100000 for CT 202's process.
- [ ] CT 201 can read `/srv/lab-root-only/host-file`; CT 202 cannot.
- [ ] Both containers show `uid=0(root)` inside — the illusion is intact.
- [ ] Cleanup done: no `/srv/lab-root-only`, no `mp0` on either container.

## Evidence to capture

- [ ] `01-config.txt` — `pct config` for 201 and 202
- [ ] `02-commands.md` — create commands and the mapping probes
- [ ] `03-output.txt` — `ps` UID output, permission denied message
- [ ] `05-verification.md` — written explanation of the mapping in your words
- [ ] `06-troubleshooting.md` — anything unexpected
- [ ] Screenshots — the `ps` output is the best single screenshot; redact PIDs
      if you prefer

## Troubleshooting log

| Symptom | Commands used | Root cause | Fix | Verified |
|---|---|---|---|---|
| Both containers succeed reading host file | `pct config 202 \| grep unprivileged` | CT 202 was created privileged by mistake | recreate with `--unprivileged 1` | |
| `ps -C sleep` shows nothing | `pct exec 202 -- ps aux` | probe command exited | re-run `systemd-run` probe | |
| Bind mount invisible after `pct set` | `pct config 202`, `findmnt` | mount changes need restart in some cases | `pct restart 202` | |
| Can't delete `mp0` | `pct config 202` | syntax | `pct set 202 --delete mp0` | |

## Key takeaways

- Unprivileged is the default you should choose, not the default you get.
- The ID map is the actual security boundary; know that 100000 is doing the work.
- A bind mount can silently reintroduce host access — mount options matter.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §5
- Next: [`05-container-networking.md`](05-container-networking.md)
