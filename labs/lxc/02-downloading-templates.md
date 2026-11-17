# Lab 02 — Downloading and Managing Templates

| Field | Value |
|---|---|
| Status | Not started |
| Track | LXC |
| Container IDs | none |
| Storage used | `local` — content type `vztmpl` |
| Network | host egress to `download.proxmox.com` |
| Prerequisites | Lab 01 |
| Estimated time | 20–30 min |
| Evidence folder | `evidence/lxc/02-downloading-templates/` |

## Concept

A container template is a compressed **userland tarball** — no kernel, no boot
loader. Proxmox manages them with `pveam` on a storage that has the `vztmpl`
content type enabled. On a default install that is `local`, physically located
at `/var/lib/vz/template/cache/`.

Templates are cached, not linked: downloading the same template twice is
wasteful, and an outdated template produces an outdated container.

## Prerequisites

- [ ] `local` storage has `vztmpl` in its content types (`cat /etc/pve/storage.cfg`).
- [ ] The node can reach `https://download.proxmox.com`.
- [ ] Free space on `local` for the template (~100–250 MB each).

## Practical implementation

### 1. Verify storage supports container templates

```bash
cat /etc/pve/storage.cfg
pvesm status
```

Look for `content ... vztmpl` (and `rootdir`) on `local`.

### 2. Test connectivity before blaming pveam

```bash
curl -sI https://download.proxmox.com | head -5
getent hosts download.proxmox.com
```

### 3. Refresh the catalogue

```bash
pveam update
```

Expected: a list of sections (`system`, `turnkeylinux`, `mail`, ...) with
counts.

### 4. Browse, then download

```bash
pveam available --section system | grep -i debian
```

Pick the exact filename from the output — template names and point releases
change. Then:

```bash
# Example only: use the filename pveam reported on your node
pveam download local debian-13-standard_13.1-1_amd64.tar.zst
```

### 5. Verify the cache

```bash
pveam list local
ls -lh /var/lib/vz/template/cache/
```

### 6. Practise lifecycle of a cached template

```bash
# Remove and re-download to prove you can manage the cache
pveam remove local:vztmpl/debian-13-standard_13.1-1_amd64.tar.zst
pveam list local
pveam download local debian-13-standard_13.1-1_amd64.tar.zst
pveam list local
```

> Keep one template cached at the end — Lab 03 needs it.

## Verification

- [ ] `pveam list local` shows the template with a plausible size.
- [ ] `ls -lh /var/lib/vz/template/cache/` shows the file on disk.
- [ ] The filename matches an entry from `pveam available --section system`.
- [ ] Architecture matches the host (`amd64` on this node — `dpkg --print-architecture`).

## Evidence to capture

- [ ] `01-config.txt` — storage config lines showing `vztmpl`
- [ ] `02-commands.md` — commands in order
- [ ] `03-output.txt` — `pveam update`, `available` (filtered), `list local`
- [ ] `05-verification.md` — filename, size, architecture justification
- [ ] `06-troubleshooting.md` — download problems if any

## Troubleshooting log

| Symptom | Commands used | Root cause | Fix | Verified |
|---|---|---|---|---|
| `no such template` | `pveam available \| grep -i debian` | stale catalogue or typo in filename | `pveam update`, copy exact name | |
| `storage 'local' does not support content type 'vztmpl'` | `cat /etc/pve/storage.cfg` | content type not enabled | enable `vztmpl` on the storage | |
| Download hangs | `curl -sI https://download.proxmox.com` | DNS / proxy / egress filter | fix host network or proxy | |
| Certificate error | `curl -v https://download.proxmox.com` | clock skew | `timedatectl`, NTP sync | |

## Key takeaways

- Templates are OS userland only — this is why architecture must match.
- `pveam available` is the source of truth for names; never trust a copied
  example filename.
- A cached template is a dependency for every later lab — keep one.

## References

- [`docs/lxc-containers.md`](../../docs/lxc-containers.md) §3
- Next: [`03-creating-first-container.md`](03-creating-first-container.md)
