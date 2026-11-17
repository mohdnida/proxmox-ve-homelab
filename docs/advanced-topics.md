# Advanced Proxmox VE Topics

> **Purpose:** The follow-on track after the ten LXC labs. Each section states
> what it is, why it matters, the implementation sketch, the risks, and the
> evidence to capture.
>
> **Important:** Several topics here require **more than one node**. This lab
> is a single nested Proxmox host, so migration, clustering and HA are
> documented as *designed and discussed*, not as *performed*. The status
> column in the [lab index](../labs/README.md) is the source of truth.

Completion order below is deliberate: it moves from container internals
outward to infrastructure.

---

## 1. LXC nesting and Docker inside LXC

**Concept.** `nesting=1` removes the restriction that prevents a container
from creating its own nested namespaces, which is what container runtimes
need. `keyctl=1` and `fuse=1` are usually required by Docker and
Podman/overlay filesystems.

**Why it matters.** Running Docker inside an LXC container is a common way to
get container-in-container isolation with far less overhead than a VM — and a
common source of confusing failures.

**Implementation sketch.**

```bash
# Unprivileged container with nesting enabled (CT 209 is reserved for this)
pct create 209 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-209-docker --unprivileged 1 --features nesting=1,keyctl=1,fuse=1 \
  --rootfs local-lvm:8 --memory 2048 --swap 512 --start 1

# Install Docker in the container (official convenience script or apt repo)
pct exec 209 -- bash -c 'command -v curl || apt-get update && apt-get install -y curl'
pct exec 209 -- bash -c 'curl -fsSL https://get.docker.com | sh'
pct exec 209 -- docker run --rm hello-world

# Inspect the resulting configuration
pct config 209 | grep features
```

**Risks and gotchas.**

- Overlayfs on top of LVM-thin often fails; `fuse-overlayfs` or the `vfs`
  storage driver is the usual workaround.
- `nesting=1` relaxes AppArmor confinement — document the decision.
- Privileged containers are **not** required for Docker; prefer unprivileged
  plus features.

**Evidence:** `pct config`, `docker info`, `docker run hello-world` output,
storage driver used, and any workaround applied.

---

## 2. Bind mounts and storage mapping

**Concept.** A bind mount exposes a host path inside the container
(`mpX /host/path,mp=/container/path`). A separate volume (`mpX <storage>:<size>`)
keeps data independent of the container root filesystem.

**Why it matters.** Data lifecycle: bind mounts let you back up or rebuild a
container without touching its data; they also let several containers share a
dataset deliberately.

**Implementation sketch.**

```bash
# Read-only host directory
pct set 204 --mp0 /srv/lab-share,mp=/mnt/share,ro=1

# Independent thin volume
pct set 204 --mp1 local-lvm:4,mp=/data

# Verify mount behaviour inside
pct exec 204 -- findmnt -T /mnt/share
pct exec 204 -- touch /mnt/share/probe   # must fail when ro=1
```

**Risks and gotchas.**

- Unprivileged containers see host files through the ID map; a host path owned
  by UID 1000 may appear owned by nobody inside. Fix host ownership or extend
  `/etc/subuid`, `/etc/subgid` and `lxc.idmap`.
- `ro=1` matters for evidence and for safety — prove the mount is read-only.

**Evidence:** `pct config`, `findmnt` output, the failed write test, and the
UID ownership observed on both sides.

---

## 3. Linux namespaces and cgroups

**Concept.** Namespaces partition visibility; cgroups partition resources.
Proxmox VE 9 uses **cgroup v2 only** — the unified hierarchy at
`/sys/fs/cgroup`.

| Namespace | Isolates |
|---|---|
| `pid` | Process IDs and process tree |
| `net` | Interfaces, routes, iptables |
| `mnt` | Mount points |
| `uts` | Hostname and domain |
| `ipc` | Shared memory, semaphores |
| `user` | UID/GID mapping |
| `cgroup` | cgroup view |
| `time` | Boot-time and monotonic clock offsets |

**Why it matters.** Every LXC symptom reduces to one of these. Once you can
inspect namespaces and cgroup files, container debugging stops being guesswork.

**Implementation sketch.**

```bash
# Host side: see the namespace set for a container's init process
CT_PID=$(pct exec 200 -- pgrep -o systemd || true)
lsns -t pid,net,mnt,user
ls -l /proc/$(pgrep -f 'lxc-start.*200' | head -1)/ns

# Cgroup resource view (v2 paths)
systemctl status pve-container@200
cat /sys/fs/cgroup/lxc/200/memory.max
cat /sys/fs/cgroup/lxc/200/cpu.max
pct exec 200 -- cat /sys/fs/cgroup/memory.max

# Compare inside vs outside
pct exec 200 -- lsns
pct exec 200 -- cat /proc/1/cgroup
```

**Risks and gotchas.** Do not attempt to modify cgroup files by hand on a
running container — use `pct set`. Manual edits are lost on restart and can
destabilise the host.

**Evidence:** `lsns` from host and container, cgroup `memory.max` / `cpu.max`
values, and the mapping back to the `pct set` options used.

---

## 4. AppArmor profiles

**Concept.** Proxmox ships AppArmor enabled by default. Each container gets a
generated profile that confines it to the paths and operations LXC expects.
Denials appear in `dmesg` and `/var/log/kern.log`.

**Why it matters.** This is the layer that stops a compromised container from
touching unrelated host files, even in privileged mode. Disabling it is
almost always the wrong fix.

**Implementation sketch.**

```bash
# Confirm AppArmor is active and profiles are loaded
cat /sys/module/apparmor/parameters/enabled
aa-status | head -30
aa-status | grep -i lxc

# Watch denials while reproducing a problem
dmesg -w | grep -i apparmor
tail -f /var/log/kern.log | grep -i apparmor

# Enable a narrowly scoped feature instead of relaxing confinement
pct set 207 --features nesting=1,keyctl=1
```

**Risks and gotchas.** Setting `lxc.apparmor.profile: unconfined` in
`/etc/pve/lxc/<CTID>.conf` is a last resort for a lab test — record it as a
deliberate, temporary exception with a rollback step.

**Evidence:** `aa-status` excerpt, the denial message, the minimal feature
change that resolved it, and confirmation the denial no longer appears.

---

## 5. Container snapshots and rollback

**Concept.** A snapshot stores container config plus filesystem state at a
point in time. Thin (LVM-thin) and ZFS storage support it; plain directory
storage does not.

**Why it matters.** It turns risky changes into reversible experiments — the
cheapest safety net available in a lab.

**Implementation sketch.**

```bash
# Thin-provisioned rootfs is required
pct snapshot 206 pre-upgrade --description "before apt upgrade"
pct listsnapshot 206

# Do the risky work, then decide
pct rollback 206 pre-upgrade
pct delsnapshot 206 pre-upgrade

# Confirm the state after rollback
pct status 206
pct exec 206 -- cat /etc/os-release
```

**Risks and gotchas.**

- Snapshot chains consume thin-pool space; a full pool breaks **every**
  container on it. Monitor `pvesm status` and delete old snapshots.
- Snapshots are not backups — they live on the same storage as the data.
- Rollback discards changes since the snapshot, including logs you may want.

**Evidence:** `pct listsnapshot` before and after, `pvesm status` free space,
and a before/after proof of the restored state.

---

## 6. Container migration between Proxmox nodes

**Concept.** `pct migrate` moves a container to another node in the same
cluster. The target must have access to the same storage or be able to
receive a streamed copy.

**Why it matters.** It is the operational reason clusters exist — maintenance
without downtime.

**Status in this lab.** **Not executable.** Requires at least two nodes with
corosync connectivity. Documented here as a design exercise only.

**Implementation sketch (for a future two-node cluster).**

```bash
pvecm status
pct status 200
pct migrate 200 pve-node2 --restart      # offline migration
pct migrate 200 pve-node2 --online       # live, if supported by the workload
```

**Risks and gotchas.** Shared storage or a fast migration network is
mandatory; nested virtualization without VT-x passthrough makes the whole
exercise slower and less representative.

**Evidence (when executed):** `pvecm status`, `pct status` before/after,
migration task log, and service availability during the move.

---

## 7. API-based container management

**Concept.** The web UI is a client of the REST API at `https://<node>:8006/api2/json`.
API tokens (`pveum`) provide scoped, revocable credentials without storing a
password.

**Why it matters.** Anything you click, you can automate — and that is how
real infrastructure is managed.

**Implementation sketch.**

```bash
# Create a dedicated user and token with a narrow role
pveum user add automation@pve --comment "automation"
pveum role add CT-Operator -privs "VM.Audit VM.PowerMgmt VM.Console"
pveum acl modify /vms/200 --users automation@pve --roles CT-Operator
pveum user token add automation@pve lab-token --privs VM.PowerMgmt

# Call the API (token secret is shown once — store it safely, never commit it)
curl -s -k -H 'Authorization: PVEAPIToken=automation@pve!lab-token=<SECRET>' \
  https://192.0.2.100:8006/api2/json/nodes/proxmox/lxc

# Power operations through the API
curl -s -k -X POST -H 'Authorization: PVEAPIToken=...' \
  https://192.0.2.100:8006/api2/json/nodes/proxmox/lxc/200/status/start
```

**Risks and gotchas.** Token secrets are credentials: never commit them, never
screenshot them, and prefer `/etc/pve/priv/` or a local `.env` that is
git-ignored. `-k` disables certificate verification — acceptable only in this
lab, document why.

**Evidence:** the `pveum` role and ACL listing (no secret values), and API
responses with tokens redacted.

---

## 8. Automation with Bash and Ansible

**Concept.** Bash for node-local glue, Ansible for idempotent, repeatable
state across hosts. Both sit on the same API/CLI surface you have already used
manually.

**Why it matters.** The ten labs are performed by hand once; automation is
what makes them reproducible and verifiable.

**Implementation sketch.**

```bash
#!/usr/bin/env bash
# create-ct.sh — idempotent container creation (example)
set -euo pipefail
CTID="${1:?CTID required}"
TEMPLATE="local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst"

if pct status "$CTID" >/dev/null 2>&1; then
  echo "CT $CTID already exists; skipping"
  exit 0
fi

pct create "$CTID" "$TEMPLATE" \
  --hostname "lab-ct-${CTID}" --unprivileged 1 \
  --cores 2 --memory 1024 --swap 512 \
  --rootfs local-lvm:8 --start 1
```

```yaml
# ansible-playbook example (community.proxmox collection)
- hosts: proxmox
  tasks:
    - name: Ensure lab container exists
      community.proxmox.proxmox:
        node: proxmox
        api_user: automation@pve
        api_token_id: lab-token
        api_token_secret: "{{ lookup('env', 'PVE_TOKEN_SECRET') }}"
        hostname: lab-ct-209
        vmid: 209
        ostemplate: local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst
        cores: 2
        memory: 1024
        unprivileged: true
        state: present
```

**Risks and gotchas.** Idempotency is the whole point: every script must be
safe to run twice. Credentials come from environment or a secret store, never
from the repository.

**Evidence:** the script itself, a dry run, a second run proving idempotency,
and the resulting `pct config`.

---

## 9. Proxmox cluster concepts and high availability

**Concept.** A cluster joins nodes with corosync for messaging and quorum,
with optional HA manager and storage replication. HA needs a quorum majority
and a fencing mechanism (watchdog) to avoid split-brain.

**Why it matters.** It is the boundary between "a lab PC running Proxmox" and
"infrastructure". Knowing the prerequisites prevents very bad designs.

**Status in this lab.** **Not executable** on a single nested node. Documented
as theory plus a future design.

**Key concepts to be able to explain.**

| Concept | Meaning |
|---|---|
| Quorum | Majority of nodes must agree; loss of quorum blocks config changes |
| Corosync | Cluster communication layer (ring 0/1 redundancy) |
| HA manager | `ha-manager` starts/moves/recovers HA resources |
| Fencing / watchdog | Prevents two nodes running the same HA resource |
| Replication | ZFS-based periodic sync between nodes |
| Expected votes | Tunable quorum threshold; avoid traps with 2-node clusters |
| Split-brain | Two partitions both believing they are authoritative — what quorum prevents |

**Implementation sketch (future two/three-node cluster).**

```bash
pvecm create lab-cluster
pvecm add 192.0.2.101
pvecm status
ha-manager status
ha-manager add ct:200 --state started
```

**Risks and gotchas.** Never build a 2-node cluster without an explicit
quorum plan. HA without fencing is worse than no HA — it can double-run a
service. Document expected votes and failure behaviour before enabling HA.

**Evidence (when executed):** `pvecm status`, `ha-manager status`, and a
documented failure test (power off one node) with the observed recovery.

---

## Reading order and prerequisites

| # | Topic | Needs |
|---|---|---|
| 1 | Nesting and Docker in LXC | labs 01–09 |
| 2 | Bind mounts and storage mapping | lab 06 |
| 3 | Namespaces and cgroups | lab 07 |
| 4 | AppArmor profiles | lab 09 |
| 5 | Snapshots and rollback | lab 08 |
| 6 | Migration | second node |
| 7 | API management | lab 03+, RBAC understanding |
| 8 | Bash and Ansible | labs 01–10 complete |
| 9 | Clustering and HA | two or three nodes |

See [`roadmap.md`](roadmap.md) for how these fit the wider Proxmox VE plan.
