# Immich Backup Restore

**Project:** Home Server  
**Trigger:** Immich data loss or VM failure

## Prerequisites
- Access to home server (SSH or console)
- Backup storage accessible

## Steps

### 1. SSH into home server
```bash
ssh user@192.168.x.x
```

### 2. Stop Immich containers
```bash
cd /opt/immich
docker compose down
```

### 3. Restore data from backup
```bash
# Restore database
pg_restore -U postgres immich_backup.dump

# Restore media files
rsync -av /backup/immich/library/ /opt/immich/library/
```

### 4. Restart and verify
```bash
docker compose up -d
# Check logs
docker compose logs -f
```

## Rollback
Restore previous backup snapshot if restore fails.
