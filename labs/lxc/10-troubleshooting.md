# Lab 10 — Troubleshooting

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | reuses CT 200–207 |
| Storage used | reuses existing |
| Network | all bridges |
| Prerequisites | Labs 01–09 |
| Estimated time | 60–90 min |
| Evidence folder | `evidence/lxc/10-troubleshooting/` |

## Concept

Troubleshooting is a procedure, not intuition. Work outward in a fixed order:

```text
config  ->  storage  ->  network  ->  AppArmor  ->  guest init  ->  resources
```

This lab injects five faults on purpose. Deliberately breaking your own lab is
the cheapest way to learn the failure signatures — and it produces the most
valuable evidence in the repository.

> **Safety:** only inject faults on containers you created for these labs.
> Never do this on a container holding data you have not backed up.

## Prerequisites

- [ ] Labs 03–09 complete, CTs 200–207 exist.
- [ ] Backup from Lab 08 available (so a destroyed container is recoverable).
- [ ] A second shell open: `journalctl -u pve-container@200 -f`.

## The first five commands, for every fault

```bash
pct status <CTID>
pct config <CTID>
pct df <CTID>
journalctl -u pve-container@<CTID> --no-pager -n 50
dmesg | tail -50
```

## Practical implementation

### Fault A — configuration parse error

```bash
cp /etc/pve/lxc/200.conf /root/200.conf.bak
echo "this_is_not_a_valid_key: 1" >> /etc/pve/lxc/200.conf
pct start 200            # or pct restart 200
```

Diagnose:

```bash
journalctl -u pve-container@200 --no-pager -n 30
pct config 200
```

Fix and verify:

```bash
cp /root/200.conf.bak /etc/pve/lxc/200.conf
pct start 200
pct status 200
```

Record the exact error text. It is the signature you will recognise next time.

### Fault B — wrong gateway

```bash
pct config 203 | grep net0
pct set 203 --net0 name=eth0,bridge=vmbr0,ip=192.0.2.203/24,gw=192.0.2.254
pct reboot 203

pct exec 203 -- ip route
pct exec 203 -- ping -c 2 192.0.2.1
```

Diagnose the difference between "local segment reachable" and "off-segment
unreachable":

```bash
pct exec 203 -- ping -c 2 192.0.2.100     # same segment may still work
pct exec 203 -- ip route get 1.1.1.1
```

Fix and verify:

```bash
pct set 203 --net0 name=eth0,bridge=vmbr0,ip=192.0.2.203/24,gw=192.0.2.1
pct reboot 203
pct exec 203 -- ping -c 2 192.0.2.1
```

### Fault C — stopped container, then a client command

```bash
pct stop 200
pct status 200
pct exec 200 -- hostname        # must fail
pct df 200                      # explains itself
```

Diagnose, fix, verify:

```bash
journalctl -u pve-container@200 --no-pager -n 20
pct start 200
pct exec 200 -- hostname
```

### Fault D — disk pressure

```bash
# Fill a bounded area inside the container, safely
pct exec 205 -- bash -c 'mkdir -p /tmp/fill; \
  dd if=/dev/zero of=/tmp/fill/blob bs=1M count=2000 status=none || true'
pct df 205
pct exec 205 -- df -h
lvs -o lv_name,lv_size,data_percent | grep -E 'vm-205|data'
```

Diagnose the difference between the container filesystem, the thin volume and
the thin pool. Then recover:

```bash
pct exec 205 -- rm -rf /tmp/fill
pct fstrim 205
pct df 205
lvs -o lv_name,data_percent | grep data
```

### Fault E — AppArmor / feature-related denial

Reproduce a denial rather than guessing one:

```bash
pct config 207 | grep features
# Start a service that needs a capability the container does not have
pct exec 207 -- bash -c 'command -v mount >/dev/null && mount -t tmpfs none /mnt 2>&1 | head -3'
dmesg | grep -i -E 'apparmor|denied' | tail -10
```

Diagnose whether the blocker is AppArmor, a missing capability, or a missing
feature. Then apply the **minimal** change — for example enabling `nesting=1`
only if the workload genuinely needs it:

```bash
pct set 207 --features nesting=1
pct reboot 207
# repeat the probe and record whether the behaviour changed
```

Roll back if it was not required:

```bash
pct set 207 --features ""
pct reboot 207
```

### Fault F — stale lock (only if you actually hit one)

```bash
pct status 200
# If it reports "locked", confirm no task is running before unlocking:
ls -lt /var/log/pve/tasks/ | head
pct unlock 200
```

Never unlock blindly: an in-flight backup interrupted by an unlock can leave a
half-written archive.

## Verification

- [ ] All five/six faults reproduced and documented with the exact signature.
- [ ] Every fault resolved and verified by an observable check.
- [ ] Container configuration restored to the pre-fault state.
- [ ] `pct list` shows all lab containers in their expected state.

## Evidence to capture

- [ ] `01-config.txt` — configs before and after each fault
- [ ] `02-commands.md` — the injected fault and the fix, in order
- [ ] `03-output.txt` — every error message, verbatim
- [ ] `04-network.txt` — route/ping evidence for Fault B
- [ ] `05-verification.md` — symptom → cause → fix → proof table
- [ ] `06-troubleshooting.md` — the consolidated runbook you would hand to
      someone else
- [ ] Screenshots — task logs showing the failure, redacted

## Diagnostic runbook (write this in your own words)

| Symptom | First check | Second check | Typical cause |
|---|---|---|---|
| `pct start` fails instantly | `journalctl -u pve-container@<id>` | `pct config <id>` | bad config line, AppArmor denial |
| `pct exec` fails | `pct status` | `journalctl` | container stopped or crashed |
| No IP inside | `pct config` | `bridge link show` | wrong net0, bridge down |
| DHCP never resolves | `ip a` in CT | DHCP server present? | no DHCP on segment |
| Gateway unreachable | `ip route` | `ip route get <addr>` | wrong gateway |
| Cross-VLAN fails | `bridge vlan show` | `bridge-vids` in interfaces | tag or VLAN awareness |
| Disk full | `pct df` | `lvs`, `pvesm status` | thin pool exhaustion |
| Random process deaths | `dmesg \| grep -i oom` | `memory.max` | memory limit |
| Permission denied on mount | `findmnt` | UID map, `/etc/subuid` | unprivileged mapping |
| `CT is locked` | `/var/log/pve/tasks/` | `pct unlock` | in-flight or stale task |

## Key takeaways

- The order matters: config → storage → network → AppArmor → init → resources.
- Every error message is evidence; copy it verbatim before "fixing" anything.
- Breaking your own lab beats reading about other people's outages.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §12
- Back up before destructive tests: [`08-container-backup-restore.md`](08-container-backup-restore.md)
