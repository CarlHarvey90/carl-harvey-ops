# VM Snapshot & Restore

**Project:** Home Server  
**Trigger:** Before major changes to any VM, or scheduled baseline snapshots

---

## When to snapshot

- Before updating a VM's OS or installed software
- Before making configuration changes to HexOS or Immich
- After initial setup of a new VM (clean baseline)
- Before running anything experimental on the Debian dev VM

---

## Create a snapshot — Proxmox UI

1. Open Proxmox web UI at `https://PROXMOX_IP:8006`
2. Select the VM from the left panel
3. Click **Snapshots** tab
4. Click **Take Snapshot**
5. Give it a meaningful name, e.g. `pre-immich-update-2026-03`
6. Optionally tick **Include RAM** if you want to snapshot running state
7. Click **Take Snapshot**

---

## Create a snapshot — CLI

SSH into the Proxmox host:

```bash
# Syntax: qm snapshot <vmid> <snapshot-name> --description "reason"
qm snapshot 101 pre-update-2026-03 --description "Before OS update"

# List snapshots for a VM
qm listsnapshot 101
```

---

## Restore a snapshot — Proxmox UI

1. Select the VM
2. Click **Snapshots** tab
3. Select the snapshot to restore
4. Click **Rollback**
5. Confirm — the VM will revert to that state

> The VM must be stopped before rolling back in most cases.

---

## Restore a snapshot — CLI

```bash
# Stop the VM first
qm stop 101

# Rollback to snapshot
qm rollback 101 pre-update-2026-03

# Start it again
qm start 101
```

---

## VM IDs — fill these in

| VM | Proxmox VMID |
|---|---|
| HexOS | — |
| Immich | — |
| OpenClaw | — |
| Debian | — |

---

## Important notes

- **Snapshots are not backups** — they live on the same storage as the VM. If the Proxmox host dies, snapshots go with it.
- For HexOS specifically, use Proxmox Backup Server or an offsite backup in addition to snapshots
- Snapshots consume disk space — delete old ones after confirming a change is stable
- For HexOS: unmount NFS shares from Immich before snapshotting to avoid inconsistent state
