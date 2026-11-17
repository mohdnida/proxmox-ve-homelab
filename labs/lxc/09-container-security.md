# Lab 09 — Container Security

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | CT 207 |
| Storage used | `local-lvm` 4 GiB |
| Network | `vmbr1` VLAN 20, static `203.0.113.20/24`, firewall enabled |
| Prerequisites | Labs 04, 05 and 06 |
| Estimated time | 45–60 min |
| Evidence folder | `evidence/lxc/09-container-security/` |

## Concept

Container security is layered, and the layers are cheap to enable **before**
the first workload arrives and painful to retrofit afterwards:

1. Unprivileged container (user namespace).
2. AppArmor confinement.
3. Minimal features (`nesting`, `keyctl`, `fuse`, `mknod` only when needed).
4. Network exposure controlled by bridge/VLAN choice plus `firewall=1`.
5. Least privilege inside the guest (no root SSH, keys not passwords).
6. Host-side RBAC, 2FA and scoped API tokens.

This lab audits all six on one container and proves each is actually on — not
just configured in a file somewhere.

## Prerequisites

- [ ] CT 207 unused.
- [ ] `vmbr1` VLAN-aware with VLAN 20 permitted.
- [ ] A host directory to test read-only access.

## Practical implementation

### 1. Create a hardened container

```bash
pct create 207 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-207-sec --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr1,tag=20,ip=203.0.113.20/24,firewall=1 \
  --unprivileged 1 \
  --features nesting=0 \
  --onboot 0 --start 1
```

Note what was **not** enabled: no `nesting`, no `keyctl`, no `fuse`. Features
are attack surface.

### 2. Audit isolation from the config

```bash
pct config 207
grep -E '^(unprivileged|lxc.idmap|features|net0)' /etc/pve/lxc/207.conf
```

Checklist:

- `unprivileged: 1` present
- exactly two `lxc.idmap` lines (u and g)
- no `features:` line, or an empty one
- `firewall=1` on `net0`
- `tag=20` with `bridge=vmbr1`

### 3. Audit AppArmor

```bash
cat /sys/module/apparmor/parameters/enabled       # expect: Y
aa-status | head -20
aa-status | grep -i lxc
dmesg | grep -i apparmor | tail -10
```

Then prove the profile applies to your container: find the container's host
process and read its AppArmor label.

```bash
pct exec 207 -- systemd-run --unit=secprobe sleep 300
PID=$(pgrep -f 'sleep 300' | head -1)
cat /proc/$PID/attr/current
```

Expected: a label referencing the container / LXC confinement rather than
`unconfined`.

### 4. Audit capabilities and identity inside

```bash
pct exec 207 -- id
pct exec 207 -- grep -E '^Cap' /proc/1/status
pct exec 207 -- cat /proc/sys/kernel/unprivileged_userns_clone 2>/dev/null || true
pct exec 207 -- ip -4 addr show eth0
```

Record the capability bitmask. You are not expected to memorise it — the point
is that it is *finite and visible*, unlike a privileged container where the
boundary is the host itself.

### 5. Read-only bind mount, and prove it

```bash
mkdir -p /srv/lab-ro
echo "read me" > /srv/lab-ro/data.txt
chmod 755 /srv/lab-ro

pct set 207 --mp0 /srv/lab-ro,mp=/mnt/ro,ro=1
pct reboot 207

pct exec 207 -- cat /mnt/ro/data.txt
pct exec 207 -- touch /mnt/ro/nope           # must fail
pct exec 207 -- findmnt -T /mnt/ro
```

### 6. Network exposure review

```bash
# What is actually listening inside
pct exec 207 -- ss -tlnp

# Firewall status at datacenter/node level
pve-firewall status
cat /etc/pve/firewall/cluster.fw 2>/dev/null | head -30
ls -l /etc/pve/firewall/
```

If you add a container-specific rule set, write it to
`/etc/pve/firewall/207.fw` (the `ct:207` section) and record the **deny test**
that proves it works — a rule set that has never blocked anything is not
verified.

Example rule set (default deny inbound, allow SSH from the lab only):

```text
[OPTIONS]
enable: 1

[IN]
DROP: all
ACCEPT: -s 192.0.2.0/24 -p tcp --dport 22
```

```bash
pve-firewall compile
pve-firewall status
# Negative test: from a host outside the allowed source, SSH must be refused
```

### 7. Host-side controls (RBAC and 2FA)

```bash
pveum user list
pveum role list | head
pveum acl list
```

- [ ] 2FA enabled for `root@pam` (Datacenter → Permissions → Two Factor).
- [ ] Any API token is scoped to a role, not full privileges.
- [ ] No shared root password used by automation.

Record only the role/ACL structure — **never** token secrets, TOTP seeds or
recovery codes.

## Verification

- [ ] `pct config` proves unprivileged + firewall + no extra features.
- [ ] `aa-status` and `/proc/<pid>/attr/current` show confinement is active.
- [ ] The read-only mount refuses writes.
- [ ] `ss -tlnp` shows only services you intend to run.
- [ ] A firewall deny test actually blocked traffic.
- [ ] 2FA is enabled for the root account.

## Evidence to capture

- [ ] `01-config.txt` — `pct config 207`, the `207.fw` file
- [ ] `02-commands.md` — audit commands in order
- [ ] `03-output.txt` — `aa-status`, capability bitmask, `ss -tlnp`
- [ ] `04-network.txt` — `ip -4 a`, firewall compile output
- [ ] `05-verification.md` — the six-point checklist with results
- [ ] `06-troubleshooting.md` — any denial and the minimal fix
- [ ] Screenshots — firewall rules and 2FA status, **redacted**

## Troubleshooting log

| Symptom | Commands used | Root cause | Fix | Verified |
|---|---|---|---|---|
| `aa-status` shows no lxc profiles | `systemctl status apparmor`, `dmesg` | AppArmor not loaded | enable/start AppArmor at boot | |
| Service denied despite correct config | `dmesg \| grep -i apparmor` | AppArmor denial | enable the specific feature, not `unconfined` | |
| Firewall rule has no effect | `pve-firewall compile`, `pve-firewall status` | firewall disabled at datacenter level | enable at the correct scope | |
| Bind mount writable despite `ro=1` | `findmnt -T`, `pct config` | mount option not applied / not restarted | `pct reboot 207` | |
| Container can still resolve external names | `ss`, `ip route` | egress not filtered by design | document the intended policy | |

## Key takeaways

- Unprivileged + no extra features is the secure default; every feature you add
  should have a written reason.
- AppArmor denials are evidence, not noise — read them before disabling
  anything.
- A firewall rule that has never blocked traffic is unverified.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §11
- [`advanced-topics.md`](../../docs/advanced-topics.md) §4 (AppArmor profiles)
- Next: [`10-troubleshooting.md`](10-troubleshooting.md)
