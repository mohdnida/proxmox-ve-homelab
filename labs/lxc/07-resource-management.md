# Lab 07 — Resource Management

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | CT 205 |
| Storage used | `local-lvm` 4 GiB |
| Network | `vmbr0`, `192.0.2.205/24` |
| Prerequisites | Lab 03 |
| Estimated time | 45–60 min |
| Evidence folder | `evidence/lxc/07-resource-management/` |

## Concept

Three knobs, three different behaviours:

| Knob | Option | Meaning |
|---|---|---|
| CPU count | `--cores` | CPUs *visible* to the container. Not a reservation. |
| CPU cap | `--cpulimit` | Hard ceiling in cores. `0` = unlimited. |
| CPU weight | `--cpuunits` | Relative share under contention (default 1024). |
| Memory | `--memory` | Hard limit in MiB (cgroup v2 `memory.max`). |
| Swap | `--swap` | Swap allowance; `0` disables swap. |

On this node there are only 2 vCPUs and ~3.8 GiB RAM, so limits are also a
safety mechanism: one runaway container can otherwise starve the web UI and
lock you out of the management interface.

## Prerequisites

- [ ] Free thin-pool space.
- [ ] `stress-ng` installable inside the container (needs egress).
- [ ] `192.0.2.205` free.

## Practical implementation

### 1. Create the container with deliberate limits

```bash
pct create 205 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-205-res --cores 2 --memory 1024 --swap 512 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=192.0.2.205/24,gw=192.0.2.1 \
  --unprivileged 1 --start 1
```

### 2. Observe the limits from inside

```bash
pct exec 205 -- nproc
pct exec 205 -- free -m
pct exec 205 -- cat /sys/fs/cgroup/memory.max
pct exec 205 -- cat /sys/fs/cgroup/cpu.max
pct exec 205 -- cat /sys/fs/cgroup/memory.swap.max
```

Record these four values as the baseline — they are the cgroup v2 view of what
`pct config` says.

### 3. Install a load generator

```bash
pct exec 205 -- apt-get update
pct exec 205 -- apt-get install -y stress-ng procps
```

### 4. Apply a CPU cap and prove it

```bash
pct set 205 --cpulimit 0.5
pct reboot 205
pct exec 205 -- cat /sys/fs/cgroup/cpu.max

# Two workers against a 0.5-core cap
pct exec 205 -- stress-ng --cpu 2 --timeout 45s --metrics-brief
```

Expected: throughput in the metrics is consistent with roughly half a core,
and host-side `top` shows the aggregate near 50 %.

```bash
# Host side while the stress test runs (second shell)
top -b -n1 | head -15
```

Then remove the cap and compare:

```bash
pct set 205 --cpulimit 0
pct reboot 205
pct exec 205 -- stress-ng --cpu 2 --timeout 45s --metrics-brief
```

### 5. Prove the memory limit

```bash
pct set 205 --memory 512 --swap 0
pct reboot 205
pct exec 205 -- cat /sys/fs/cgroup/memory.max
pct exec 205 -- free -m

# Try to allocate more than the limit
pct exec 205 -- stress-ng --vm 2 --vm-bytes 400M --timeout 30s
```

Expected: the OOM killer fires. Evidence:

```bash
dmesg | grep -i -E 'oom|killed process' | tail -5
pct exec 205 -- dmesg 2>/dev/null | tail -5
pct status 205
```

Record whether the container itself survived (it should) and which process was
killed. That distinction is the learning point.

### 6. Compare `cores` vs `cpulimit`

```bash
pct set 205 --cores 1 --memory 1024 --swap 512 --cpulimit 0
pct reboot 205
pct exec 205 -- nproc
pct exec 205 -- cat /sys/fs/cgroup/cpu.max
```

`nproc` drops to 1, but the container is still not *reserved* anything. Explain
in your evidence why `--cores 4` on a 2-vCPU host is legal and misleading.

## Verification

- [ ] `cpu.max` reflects `--cpulimit` after reboot.
- [ ] Stress test throughput visibly differs between capped and uncapped runs.
- [ ] `memory.max` matches `--memory`.
- [ ] An OOM event is captured in `dmesg` when the limit is exceeded.
- [ ] You can distinguish "visible CPUs" from "guaranteed CPU".

## Evidence to capture

- [ ] `01-config.txt` — `pct config 205` at each stage
- [ ] `02-commands.md` — commands in order, including both stress runs
- [ ] `03-output.txt` — `stress-ng` metrics before/after the cap
- [ ] `05-verification.md` — the four checks plus your explanation
- [ ] `06-troubleshooting.md` — unexpected behaviour
- [ ] Screenshots — host `top` during the capped run

## Troubleshooting log

| Symptom | Commands used | Root cause | Fix | Verified |
|---|---|---|---|---|
| `cpu.max` unchanged after `pct set` | `pct config`, `cat cpu.max` | not restarted | `pct reboot 205` | |
| No OOM event | `dmesg`, `free -m` | allocation below limit | raise `--vm-bytes` above `memory.max` | |
| `stress-ng` not found | `apt-get install` | package not installed | install it | |
| Host becomes unresponsive | `top` on host | no cap on a 2-vCPU host | set `--cpulimit`, test one at a time | |
| Stress test killed instantly | `dmesg` | swap disabled + tight memory limit | expected; record it | |

## Key takeaways

- `--cores` is a view, `--cpulimit` is a limit, `--cpuunits` is a priority.
- Memory limits are enforced by cgroup v2 and manifest as OOM kills.
- `--swap 0` turns "slow" into "died" — sometimes exactly what you want.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §10
- Next: [`08-container-backup-restore.md`](08-container-backup-restore.md)
