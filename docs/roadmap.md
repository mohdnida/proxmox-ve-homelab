# Proxmox VE Learning Roadmap

> **Purpose:** The overall study plan for this lab, beyond LXC. LXC has its own
> concept reference ([`lxc-containers.md`](lxc-containers.md)), lab track
> ([`labs/lxc/`](../labs/lxc/)) and follow-on ideas
> ([`advanced-topics.md`](advanced-topics.md)).
>
> **Rule for every area:** Concept → Practical implementation → Evidence and
> troubleshooting. A topic is only "done" when it has a verification step and
> captured evidence.

---

## Areas and topics

| Area | Topics to cover | Practical target | Evidence |
|---|---|---|---|
| **Networking** | Linux bridges, VLAN tagging, bonds, SDN | Extend `vmbr0`/`vmbr1`, tag VLAN 10/20, add a bond in a nested-safe way | `ip -br link`, `bridge vlan show`, `/etc/network/interfaces`, ping matrix |
| **Storage** | ZFS, LVM-thin, NFS, CIFS, Ceph | Thin pool sizing, add an NFS/CIFS share, evaluate ZFS on a spare disk | `pvesm status`, `lvs`, `zpool status`, `zfs list`, mount output |
| **Security** | RBAC, permissions, 2FA, firewall | Two roles, one scoped API token, 2FA on the root account, one firewall rule set | `pveum role list`, `pveum acl list`, firewall rules, failed-access test |
| **Backup** | PBS, retention, restore testing | `vzdump` job with retention, restore to a new ID, evaluate Proxmox Backup Server | task logs, archive listing, restore proof, retention config |
| **Automation** | Proxmox API, CLI, Ansible | Token-based API call, idempotent Bash script, one Ansible task | API response (redacted), script re-run proving idempotency |
| **Administration** | Updates, repositories, logs, services | Update the node, review repositories, inspect journal and `pveproxy` service | `apt` output, repo list, `journalctl` excerpt, `systemctl status` |
| **Advanced** | Clustering, HA, migration, replication | Design exercise on paper; execute only with a second node | `pvecm status`, `ha-manager status`, migration task log |

---

## Phases

### Phase 0 — Baseline (complete)

- Proxmox VE 9.2.2 installed nested in VMware Workstation.
- Node `proxmox` reachable at `192.0.2.100/24` on `vmbr0`.
- `vmbr1` created as an isolated, VLAN-aware bridge permitting VLAN 10 and 20.
- `local` (directory) and `local-lvm` (LVM-thin) storage in use.
- VM 100 created; blocked by unavailable nested KVM.
- Known constraints documented: ~3.8 GiB RAM, 20 GB disk, no nested VT-x.

### Phase 1 — Containers (current)

Ten labs in [`labs/lxc/`](../labs/lxc/), from architecture through
troubleshooting. This is the main deliverable of the current phase and the
foundation for everything after it.

Exit criteria:

- [ ] All ten labs executed and documented.
- [ ] Evidence folder populated for at least labs 03, 05, 08 and 10.
- [ ] One container destroyed and restored from backup successfully.

### Phase 2 — Storage and backup depth

- Thin pool capacity and monitoring; what happens when it fills.
- ZFS concepts (`ashift`, datasets, `zpool status`, scrubs) — on a spare disk.
- NFS and CIFS shares, including the `pvesm` content types.
- Backup retention policy and a scheduled job.
- Restore drill: prove a container and a VM both come back.

### Phase 3 — Networking depth

- Bridge VLAN semantics: `bridge-vlan-aware`, `bridge-vids`, access vs trunk.
- Bonding (active-backup at minimum) and why it may not work under VMware NAT.
- SDN overview: zones, VNets, subnets; evaluate whether it fits a single node.
- Segment-to-segment ping matrix as evidence.

### Phase 4 — Security and identity

- RBAC roles and ACLs on `/vms`, `/storage`, `/nodes`.
- API tokens with least privilege; token rotation.
- 2FA on the root account, then on a delegated admin.
- Firewall at datacenter, node and VM/CT level — order of evaluation.
- Document one deliberate deny test that proves the rule works.

### Phase 5 — Automation

- Bash helpers for create/start/backup with idempotency.
- API calls with `curl` using a scoped token.
- Ansible with `community.proxmox` for container lifecycle.
- Optional: a small script that verifies lab state and prints a report.

### Phase 6 — Cluster and HA (requires more hardware)

- Three-node design (or nested nodes if resources allow).
- Quorum and expected votes rationale.
- HA resources, watchdog/fencing, and a documented failure test.
- Migration and replication.

---

## Progress tracker

| Area | Concept notes | Practical lab | Evidence | Status |
|---|---|---|---|---|
| LXC containers | [`lxc-containers.md`](lxc-containers.md) | [`labs/lxc/`](../labs/lxc/) | pending | In progress |
| Advanced container topics | [`advanced-topics.md`](advanced-topics.md) | — | pending | Not started |
| Networking | [`proxmox-learning-notes.md`](proxmox-learning-notes.md) §14 | pending | partial | In progress |
| Storage | [`proxmox-learning-notes.md`](proxmox-learning-notes.md) §5, §18 | pending | partial | In progress |
| Firewall | [`proxmox-learning-notes.md`](proxmox-learning-notes.md) §15 | pending | none | Discussed only |
| Backup | [`proxmox-learning-notes.md`](proxmox-learning-notes.md) §16 | pending | none | Discussed only |
| Security / RBAC / 2FA | this document | pending | none | Not started |
| Automation | [`advanced-topics.md`](advanced-topics.md) §7–8 | pending | none | Not started |
| Cluster / HA | [`advanced-topics.md`](advanced-topics.md) §6, §9 | blocked | none | Blocked (single node) |

Status values: **Not started**, **In progress**, **Discussed only**,
**Blocked**, **Done**.

---

## Definition of done

A topic counts as complete only when all four are true:

1. The concept is explained in your own words, with the limits stated.
2. It was implemented on the lab node and the exact commands are recorded.
3. Output or screenshots prove the result — with secrets redacted.
4. The troubleshooting section records what actually went wrong.

Anything less is reading, not learning. Label it honestly.
