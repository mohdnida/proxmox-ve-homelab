# Lab 05 — Container Networking

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | CT 203 (plus CT 200 as a test peer) |
| Storage used | `local-lvm` 4 GiB |
| Network | `vmbr0` static + DHCP test, `vmbr1` VLAN 10 / VLAN 20 |
| Prerequisites | Lab 03, VLAN-aware `vmbr1` already configured |
| Estimated time | 60–75 min |
| Evidence folder | `evidence/lxc/05-container-networking/` |

## Concept

Proxmox pairs a `veth` interface inside the container with a port on a Linux
bridge. The bridge is a virtual switch: what a VM gets from a physical NIC, a
container gets from this pair.

Three things must line up for traffic to flow:

1. The bridge exists and is up.
2. The container has an address and gateway that match the segment.
3. For VLANs, the bridge is VLAN-aware **and** the tag is in the permitted list.

This lab proves each one separately so a future failure can be localised
without guessing.

## Prerequisites

- [ ] `vmbr1` is VLAN-aware with VLANs 10 and 20 permitted
      (`cat /etc/network/interfaces`, `bridge vlan show`).
- [ ] Management bridge `vmbr0` is up on `192.0.2.0/24`.
- [ ] `192.0.2.203` free; `198.51.100.10` and `203.0.113.20` reserved for this lab.

> **Warning:** never change `vmbr0` addressing without console access. A
> mistake there cuts the web UI.

## Practical implementation

### 1. Baseline the bridges from the host

```bash
ip -br link show
bridge link show
bridge vlan show
cat /etc/network/interfaces
```

### 2. Create the container with a static address

```bash
pct create 203 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-203-net --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=192.0.2.203/24,gw=192.0.2.1 \
  --unprivileged 1 --start 1
```

### 3. Verify static addressing end to end

```bash
pct exec 203 -- ip -4 addr show eth0
pct exec 203 -- ip route
pct exec 203 -- ping -c 3 192.0.2.1
pct exec 203 -- ping -c 3 192.0.2.100
bridge link show | grep -E 'veth|fwbr' | head
```

Record the host-side `veth` name that corresponds to CT 203 — it is useful
evidence that the container is bridged, not NATed.

### 4. Switch to DHCP and observe the difference

```bash
pct set 203 --net0 name=eth0,bridge=vmbr0,ip=dhcp
pct reboot 203
pct exec 203 -- ip -4 addr show eth0
pct exec 203 -- ip route
```

If no DHCP server exists on `192.0.2.0/24` the interface will stay without an
address — that is a **result**, record it, then restore static addressing:

```bash
pct set 203 --net0 name=eth0,bridge=vmbr0,ip=192.0.2.203/24,gw=192.0.2.1
pct reboot 203
```

### 5. VLAN 10 on the isolated bridge

```bash
pct set 203 --net0 name=eth0,bridge=vmbr1,tag=10,ip=198.51.100.10/24
pct reboot 203

pct exec 203 -- ip -4 addr show eth0
pct exec 203 -- ip route
bridge vlan show | grep -A1 vmbr1
```

Expected: the interface carries `198.51.100.10/24`, and `bridge vlan show`
lists the container's port under VLAN 10.

### 6. Prove segmentation with a second container on VLAN 20

```bash
pct create 210 local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst \
  --hostname lab-ct-210-vlan20 --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:2 \
  --net0 name=eth0,bridge=vmbr1,tag=20,ip=203.0.113.20/24 \
  --unprivileged 1 --start 1

# Different VLAN: must fail
pct exec 203 -- ping -c 2 203.0.113.20

# Prove the tag is applied on both ports
bridge vlan show | grep -E 'veth'
```

Optionally move 210 to VLAN 10 and repeat the ping — it should then succeed,
which proves the tag is what separates them.

### 7. Restore the lab baseline

```bash
pct set 203 --net0 name=eth0,bridge=vmbr0,ip=192.0.2.203/24,gw=192.0.2.1
pct reboot 203
pct destroy 210 --purge
```

## Verification

- [ ] Static address and gateway are correct inside CT 203.
- [ ] Ping to `192.0.2.1` succeeds; ping across VLAN 10 → 20 fails.
- [ ] `bridge vlan show` proves the tag per port.
- [ ] You can name the host `veth` interface for the container.
- [ ] DHCP behaviour was recorded, not guessed.

## Evidence to capture

- [ ] `01-config.txt` — `pct config 203` (static, DHCP, VLAN states)
- [ ] `02-commands.md` — every `pct set --net0` variant used
- [ ] `03-output.txt` — ping results, including the expected failure
- [ ] `04-network.txt` — `ip -br a`, `ip route`, `bridge link`, `bridge vlan`
- [ ] `05-verification.md` — the ping matrix
- [ ] `06-troubleshooting.md` — problems and fixes
- [ ] Screenshots — bridge VLAN view and the failed cross-VLAN ping

## Ping matrix template

| Source | Target | Expected | Observed |
|---|---|---|---|
| CT 203 `192.0.2.203` | gateway `192.0.2.1` | success | |
| CT 203 `192.0.2.203` | host `192.0.2.100` | success | |
| CT 203 VLAN 10 | CT 210 VLAN 20 | failure | |
| CT 203 VLAN 10 | CT 210 VLAN 20 (after retag) | success | |
| CT 203 | internet (if allowed) | depends on NAT | |

## Troubleshooting log

| Symptom | Commands used | Root cause | Fix | Verified |
|---|---|---|---|---|
| No address after `pct set` | `pct config 203`, `ip a` | change applied but not rebooted | `pct reboot 203` | |
| Ping gateway fails | `ip route`, `bridge link show` | wrong gateway or bridge down | correct `--net0` | |
| VLAN traffic silently dropped | `bridge vlan show`, `cat /etc/network/interfaces` | bridge not VLAN-aware / VLAN not permitted | enable VLAN awareness, add `bridge-vids` | |
| Both containers reach each other despite tags | `bridge vlan show` | `tag=` missing on one side | set the tag explicitly | |
| DHCP never resolves | `pct exec -- ip a`, check DHCP server | no server on segment | use static, document it | |

## Key takeaways

- Bridge, address, and tag are three separate checks — test them separately.
- A "failed" ping proves segmentation only when the same pair succeeds when
  retagged.
- `bridge vlan show` is the authoritative VLAN evidence on the host.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §6
- [`proxmox-learning-notes.md`](../../docs/proxmox-learning-notes.md) §14
- Next: [`06-container-storage.md`](06-container-storage.md)
