# LXC Evidence

Evidence for the ten labs in [`labs/lxc/`](../../labs/lxc/). One folder per lab,
same numbering as the lab file.

## Folder convention

```text
evidence/lxc/
├── 01-understanding-lxc/
├── 02-downloading-templates/
├── 03-creating-first-container/
├── 04-privileged-vs-unprivileged/
├── 05-container-networking/
├── 06-container-storage/
├── 07-resource-management/
├── 08-container-backup-restore/
├── 09-container-security/
└── 10-troubleshooting/
```

Empty folders are not tracked by git. Create a folder when you populate it, or
add a placeholder `README.md` inside it with the lab name and date.

## File naming

| File | Contents |
|---|---|
| `01-config.txt` | `pct config <id>` output, timestamped |
| `02-commands.md` | Commands executed, in order, with purpose |
| `03-output.txt` | Raw output of verification commands |
| `04-network.txt` | `ip -br a`, `ip route`, `bridge link`, `bridge vlan show` |
| `05-verification.md` | The proof that the lab objective was met |
| `06-troubleshooting.md` | Symptom → diagnosis → fix → verification |
| `*.png` / `*.jpg` | Screenshots, redacted before committing |

Prefix files with the lab number so ordering is stable, and keep raw
`.txt` output unedited — it is the evidence, not a summary.

## What makes good evidence

- **Timestamped.** Note the date and Proxmox version. Config changes over time.
- **Before and after.** A config dump after the change proves nothing without
  the starting state.
- **Reproducible.** Commands exactly as typed, including the ones that failed.
- **Verifiable.** Each claim has a command whose output supports it.
- **Honest.** If a step was skipped or blocked, say so.

## Redaction checklist — run before every commit

- [ ] No passwords, tokens, or token secrets anywhere in text or screenshots.
- [ ] No SSH private keys or certificate private material.
- [ ] No real public IPs; documentation ranges only (`192.0.2.0/24`,
      `198.51.100.0/24`, `203.0.113.0/24`).
- [ ] No real hostnames, domains, or email addresses you do not want public.
- [ ] Screenshots: crop or blur dashboard usernames, browser tabs, task logs
      containing paths with personal names, and the top-right account label.
- [ ] No MAC addresses or serial numbers from real hardware.
- [ ] No VM/CT names that reveal client or employer names.

If in doubt, leave it out. Absence is not a gap; a leak is permanent and
git history keeps it even after you delete the file.

## Note on deleted screenshots

Git keeps deleted blobs. Removing a sensitive screenshot in a later commit does
**not** remove it from history. If something sensitive is committed by
mistake, treat the credential as compromised, rotate it, and rewrite history
deliberately — do not assume a delete is enough.
