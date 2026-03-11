# HexOS NFS Share Recovery

**Project:** Home Server  
**Trigger:** Immich cannot access photos, NFS mount dropped, or HexOS went down

---

## Symptoms
- Immich shows no photos or library error
- NFS mount on Immich VM shows as unavailable
- HexOS VM was restarted or went down

---

## Step 1 — Verify HexOS is running

In Proxmox UI or via SSH to Proxmox host:

```bash
# Check VM status in Proxmox
qm status HEXOS_VMID
```

If stopped:
```bash
qm start HEXOS_VMID
```

Wait for HexOS to fully boot before proceeding — TrueNAS-based systems can take a minute to start NFS services.

---

## Step 2 — Verify NFS share is active on HexOS

SSH into HexOS or open the web UI and confirm:
- The NFS share is enabled
- The dataset/pool is mounted and healthy
- No storage errors in the dashboard

---

## Step 3 — Check NFS mount on Immich VM

SSH into the Immich VM:

```bash
ssh user@IMMICH_IP

# Check if the NFS share is mounted
df -h | grep photos

# Or check mount table
mount | grep nfs
```

If the share is not mounted:

```bash
# Remount manually (update with your real mount path and HexOS IP)
sudo mount -t nfs HEXOS_IP:/mnt/photos /mnt/photos
```

---

## Step 4 — Make mount persistent

If the NFS mount is not in `/etc/fstab`, it will drop after every reboot. Add it:

```bash
sudo nano /etc/fstab
```

Add a line like:
```
HEXOS_IP:/mnt/photos  /mnt/photos  nfs  defaults,_netdev  0  0
```

Test it without rebooting:
```bash
sudo mount -a
```

---

## Step 5 — Restart Immich if needed

If Immich was running when the NFS mount dropped, it may need a restart:

```bash
cd /path/to/immich
docker compose restart
```

Then verify the library is accessible in the Immich web UI.

---

## Notes
- HexOS is a dependency of Immich — always start HexOS first
- NFS drops are usually caused by HexOS rebooting — the `_netdev` fstab flag helps handle this gracefully
- Do not snapshot HexOS while the NFS share is actively mounted on Immich — unmount first
