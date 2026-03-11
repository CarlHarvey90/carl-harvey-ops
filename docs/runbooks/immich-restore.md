# Immich Recovery

**Project:** Home Server  
**Trigger:** Immich web UI unreachable, photos not loading, or VM failure

---

## Immich web UI not loading

### 1. Check the VM is running in Proxmox
```bash
qm status IMMICH_VMID
```

If stopped:
```bash
qm start IMMICH_VMID
```

### 2. SSH into Immich VM and check Docker stack
```bash
ssh user@IMMICH_IP
cd /path/to/immich
docker compose ps
```

All services should show `Up`. Key containers:
- `immich_server`
- `immich_microservices`
- `immich_machine_learning`
- `database` (PostgreSQL)
- `redis`

### 3. Restart the stack if containers are down
```bash
docker compose up -d
docker compose logs -f
```

---

## Photos not showing / library empty

This is almost always an NFS issue. See [HexOS NFS Recovery](hexos-nfs-recovery.md).

Quick check:
```bash
# On Immich VM — is the photo library mount available?
ls /mnt/photos
```

If empty or errors, the NFS mount from HexOS has dropped.

---

## Full VM restore from Proxmox snapshot

If the Immich VM is corrupted or unrecoverable:

```bash
# On Proxmox host
qm stop IMMICH_VMID
qm rollback IMMICH_VMID snapshot-name
qm start IMMICH_VMID
```

After restore, verify:
1. NFS mount to HexOS is working
2. Docker stack is running
3. Web UI is accessible at `http://IMMICH_IP:2283`

---

## Notes

- Photos themselves live on HexOS — restoring the Immich VM does not risk photo data loss
- The Immich **database** (albums, faces, metadata, user accounts) lives inside the Immich VM — this should be backed up separately
- Database backup command:
```bash
docker compose exec database pg_dumpall -U postgres > immich-db-backup-$(date +%Y%m%d).sql
```
- Store database backups on HexOS, not on the Immich VM itself
