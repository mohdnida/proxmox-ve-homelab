# Labs

Practical exercises for this Proxmox VE lab. Each lab follows the same
structure so results are comparable and the repository stays a portfolio
rather than a notebook.

## The three-part rule

Every lab is built around:

1. **Concept** — what it is, why it exists, what its limits are.
2. **Practical implementation** — exact commands, in order, on this lab node.
3. **Evidence and troubleshooting** — what to capture, what breaks, how it was
   fixed.

## Directory layout

```text
labs/
├── README.md              <- this file: index, conventions, evidence standard
├── _lab-template.md       <- copy this to start a new lab
└── lxc/
    ├── 01-understanding-lxc.md
    ├── 02-downloading-templates.md
    ├── 03-creating-first-container.md
    ├── 04-privileged-vs-unprivileged.md
    ├── 05-container-networking.md
    ├── 06-container-storage.md
    ├── 07-resource-management.md
    ├── 08-container-backup-restore.md
    ├── 09-container-security.md
    └── 10-troubleshooting.md
```

## Lab index

| # | Lab | Container IDs | Storage | Network |
|---|---|---|---|---|
| 01 | Understanding LXC | inspect only | — | — |
| 02 | Downloading templates | — | `local` (`vztmpl`) | host egress |
| 03 | Creating first container | CT 200 | `local-lvm` | `vmbr0` static |
| 04 | Privileged vs unprivileged | CT 201, CT 202 | `local-lvm` | `vmbr0` |
| 05 | Container networking | CT 203 + CT 210 (VLAN peer) | `local-lvm` | `vmbr0`, `vmbr1` VLAN 10/20 |
| 06 | Container storage | CT 204 | `local-lvm` + bind mount | `vmbr0` |
| 07 | Resource management | CT 205 | `local-lvm` | `vmbr0` |
| 08 | Backup and restore | CT 206 → restore CT 208 | `local`, `local-lvm` | `vmbr0` |
| 09 | Container security | CT 207 | `local-lvm` | `vmbr1` VLAN 20 |
| 10 | Troubleshooting | reuse 200–207 | all | all |

Container IDs 200–210 are reserved for labs so they never collide with VM 100.
CT 210 is a temporary VLAN peer (Lab 05), CT 208 a temporary restore target
(Lab 08), and CT 209 is reserved for the Docker/nesting exercise in
[`docs/advanced-topics.md`](../docs/advanced-topics.md).

## Conventions

- **Sanitised addressing only.** Documentation ranges: `192.0.2.0/24`
  (management), `198.51.100.0/24` (VLAN 10, LAB-SERVERS), `203.0.113.0/24`
  (VLAN 20, LAB-CLIENTS). Never publish real lab addresses.
- **Node name** `proxmox`, **hostname pattern** `lab-ct-<CTID>`.
- **Status honesty.** Every lab has a `Status` field. Use
  `Not started`, `In progress`, `Done`, or `Blocked` (with the reason). A lab
  is not `Done` because the notes are written — only because the steps ran and
  evidence exists.
- **One variable at a time.** Change one thing, verify, record, then continue.
- **Snapshots before risky steps.** Thin storage supports them; use them.

## Evidence standard

Every lab that changes state must produce evidence in
`evidence/lxc/<lab-number>-<slug>/`:

| Evidence | Why |
|---|---|
| `01-config.txt` — `pct config <id>` output | The single most useful debugging artefact |
| `02-commands.md` — commands run, in order | Reproducibility |
| `03-output.txt` — relevant command output | Proof, not assertion |
| `04-network.txt` — `ip a`, `ip route`, bridge state | Network labs specifically |
| `05-verification.md` — the check that proves success | Distinguishes "did" from "worked" |
| `06-troubleshooting.md` — problem, diagnosis, fix | The most reusable content |
| Screenshots — dashboard, storage, network | Visual proof; **redact first** |

See [`evidence/lxc/README.md`](../evidence/lxc/README.md) for the redaction
checklist before committing anything.

### Never commit

Passwords, API tokens, token secrets, SSH private keys, certificates with
private material, real public IPs, internal hostnames you do not want public,
unredacted screenshots (browser tabs, dashboard usernames, email addresses),
or `.dev.vars`-style credential files.

## Safety

- `pct destroy` and `pct rollback` are irreversible — back up or snapshot first.
- Networking changes on `vmbr0` can cut your management access. Have console
  access before touching it.
- Watch free space on `local-lvm`. A full thin pool affects **all** containers.
- The lab runs nested without VT-x passthrough, so VM-based work is slower and
  some features are unavailable. Do not mistake an environment limitation for
  a configuration error.

## Adding a new lab

1. Copy [`_lab-template.md`](_lab-template.md) to `labs/<track>/NN-name.md`.
2. Fill in the metadata table, especially `Status` and `Prerequisites`.
3. Add a row to the index above.
4. Create the matching `evidence/<track>/NN-name/` folder.
5. Update the status in [`docs/roadmap.md`](../docs/roadmap.md).
