# VM Snapshot & Restore

**Project:** Home Server  
**Trigger:** Before major changes or for scheduled backups

## Prerequisites
- Hypervisor access (Proxmox / ESXi)

## Snapshot

### Proxmox
```bash
# Create snapshot
qm snapshot <VMID> <snapshot-name> --description "Pre-update snapshot"

# List snapshots
qm listsnapshot <VMID>
```

## Restore

### Proxmox
```bash
# Rollback to snapshot
qm rollback <VMID> <snapshot-name>
```

## Notes
- Snapshots are not backups — use full backups for disaster recovery
- Delete old snapshots to reclaim disk space
