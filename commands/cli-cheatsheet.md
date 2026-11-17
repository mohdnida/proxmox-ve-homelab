# Proxmox VE CLI Cheat Sheet

Commands below are reference examples. Verify VM IDs, storage names and paths before running them.

## Host and version

```bash
pveversion -v
cat /etc/os-release
hostname -I
lscpu
free -h
lsblk
pvesm status
```

## Storage

```bash
df -h
lvs
cat /etc/pve/storage.cfg
pvesm list local
pvesm list local --content backup
```

## Network

```bash
ip -br addr
ip -br link
ip route
cat /etc/network/interfaces
bridge link
bridge vlan show
ip -d link show vmbr1
ping -c 4 <gateway-ip>
ping -c 4 8.8.8.8
```

## Virtual machines

```bash
qm list
qm status <VMID>
qm config <VMID>
qm start <VMID>
qm shutdown <VMID>
qm stop <VMID>
qm reboot <VMID>
qm clone <SOURCE-VMID> <NEW-VMID> --name <NAME> --full 1
qm template <VMID>
```

## Snapshots

```bash
qm snapshot <VMID> <SNAPSHOT-NAME>
qm listsnapshot <VMID>
qm rollback <VMID> <SNAPSHOT-NAME>
qm delsnapshot <VMID> <SNAPSHOT-NAME>
```

## Backup and restore

```bash
vzdump <VMID> --storage <STORAGE> --mode snapshot
pvesm list <STORAGE> --content backup
qmrestore <BACKUP-FILE> <NEW-VMID> --storage <TARGET-STORAGE>
pvesh get /cluster/backup
cat /etc/pve/jobs.cfg
```

## Firewall inspection

```bash
pve-firewall status
systemctl status pve-firewall --no-pager
pve-firewall compile
pve-firewall localnet
pvesh get /cluster/firewall/options
pvesh get /nodes/<NODE>/firewall/options
```

## Nested virtualization diagnostics

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
ls -l /dev/kvm
```

## Caution

- `qm stop`, `qm reset`, `qm destroy`, firewall policy changes and network reloads can have disruptive effects.
- Do not apply example firewall policies to a management interface without allowing required management traffic.
- Keep VMware console access available when changing Proxmox networking.
