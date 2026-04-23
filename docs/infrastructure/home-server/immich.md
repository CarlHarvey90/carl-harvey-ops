# Immich Ops Runbook

## 1. Update Immich (Docker Compose)

### Prerequisites
- SSH access to the Immich VM
- Docker and Docker Compose installed

### Steps

**1. Back up the database**
```bash
docker exec -t immich_postgres pg_dumpall -c -U postgres > immich_backup_$(date +%Y%m%d).sql
```
Backup is saved to the current working directory (e.g. `/root/`).

To save to a specific location:
```bash
docker exec -t immich_postgres pg_dumpall -c -U postgres > /path/to/backup/immich_backup_$(date +%Y%m%d).sql
```

**2. Check release notes**

Review breaking changes at: https://github.com/immich-app/immich/releases

**3. Check if IMMICH_VERSION is pinned**
```bash
grep IMMICH_VERSION /path/to/immich/.env
```

- If pinned to a specific version (e.g. `v2.3.0`), update it manually before pulling.
- If set to `v2` (major version tag), no change needed.

**4. Pull latest images and restart**
```bash
cd /path/to/immich
docker compose pull
docker compose up -d
```

**5. Clean up old images**
```bash
docker image prune
```

### Notes
- Immich follows semantic versioning. Using `:v2` as the version tag is recommended — it pulls the latest v2 release without risking major version breaking changes.
- After update, if the logs show `Reindexing clip_index` or `Reindexing face_index` for an extended period, this is normal — allow it to complete.

---

## 2. Import Google Photos Takeout into Immich

### Prerequisites
- Immich CLI installed (`npm install -g @immich/cli`)
- immich-go installed (`/usr/local/bin/immich-go`)
- Access to the Google Takeout zip file
- Immich API key (generated via **Account Settings → API Keys** in the Immich web UI)
- NAS share mounted (if takeout is stored on NAS)

### Steps

**1. Mount NAS share (if needed)**
```bash
sudo mkdir -p /mnt/nas-photos
sudo mount -t cifs //[NAS-IP]/[SHARE-NAME] /mnt/nas-photos -o username=[NAS-USER],password=[NAS-PASSWORD],vers=3.0
```

Verify the takeout file is accessible:
```bash
ls /mnt/nas-photos/[PATH-TO-TAKEOUT]/
```

**2. Log in to Immich CLI**
```bash
immich login http://localhost:2283 [API-KEY]
```

**3. Run the Google Photos import using immich-go**
```bash
immich-go upload from-google-photos \
  --server http://localhost:2283 \
  --api-key [API-KEY] \
  /mnt/nas-photos/[PATH-TO-TAKEOUT]/[TAKEOUT-FILENAME].zip
```

immich-go reads the zip directly — no need to extract first.

**4. Unmount NAS share when done**
```bash
sudo umount /mnt/nas-photos
```

**5. Verify import**
- Check the Immich web UI to confirm photos appear with correct dates.
- Navigate to **Administration → Jobs** to monitor thumbnail generation.
- If thumbnails show "Error loading image", allow background jobs to complete — this is normal post-import.

### Installing immich-go (if not already installed)
```bash
wget https://github.com/simulot/immich-go/releases/latest/download/immich-go_Linux_x86_64.tar.gz
tar -xzf immich-go_Linux_x86_64.tar.gz
sudo mv immich-go /usr/local/bin/
immich-go --version
```

### Installing Immich CLI (if not already installed)
```bash
sudo apt update
sudo apt install -y nodejs npm
sudo npm install -g @immich/cli
```
