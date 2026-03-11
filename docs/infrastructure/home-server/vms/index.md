# VM Details

## HexOS

**Base:** TrueNAS-based (HexOS)  
**Purpose:** Central personal file storage and NFS share provider  
**Network:** Local network  
**Status:** 🟢 Running — critical, keep online

### Role
- Stores all personal files and documents
- Serves photo library to Immich via NFS share
- Primary persistent storage for the home server

### Key Configuration
| Setting | Value |
|---|---|
| IP | HEXOS_IP |
| Web UI | http://HEXOS_IP |
| NFS share path | /mnt/photos (or update with real path) |
| NFS consumers | Immich VM |

### Notes
- HexOS must be running before Immich starts
- Do not snapshot while NFS share is actively mounted — unmount from Immich first
- Treat as the most critical VM on the server — data lives here

---

## Immich

**Base:** Linux  
**Purpose:** Self-hosted Google Photos alternative  
**Network:** Local network  
**Status:** 🟢 Running

### Role
- Web UI for browsing, searching, and managing personal photo library
- Reads photo library from HexOS NFS share
- Does **not** store photos itself — source of truth is HexOS

### Key Configuration
| Setting | Value |
|---|---|
| IP | IMMICH_IP |
| Web UI | http://IMMICH_IP:2283 |
| Photo source | NFS mount from HexOS |
| Stack | Docker Compose |

### Dependencies
- Requires HexOS NFS share to be mounted and available
- If HexOS goes down, Immich loses access to the photo library but the app itself stays up

### Notes
- Immich indexes photos from the NFS mount — do not move or rename the source folder without re-indexing
- Backups: the Immich database (faces, albums, metadata) should be backed up separately from the photos themselves

---

## OpenClaw

**Base:** Linux  
**Purpose:** OpenClaw bot  
**Network:** Fully isolated  
**Status:** 🟢 Running

### Role
- Runs the OpenClaw bot in a controlled, isolated environment
- Intentionally has no access to the local network or internet

### Network Isolation
| Access type | Status |
|---|---|
| SSH inbound | Allowed |
| Local network | Blocked |
| Internet outbound | Blocked |
| All other inbound | Blocked |

### Access
```bash
ssh user@OPENCLAW_IP
```

### Notes
- Isolation is intentional — verify firewall rules at the Proxmox/network level, not just inside the VM
- If OpenClaw ever needs internet access, treat that as a significant architecture decision and document it here
- Keep SSH key access only — no password auth

---

## Debian (Dev / Testing)

**Base:** Debian  
**Purpose:** General purpose development and testing  
**Network:** Local network  
**Status:** 🟡 On demand

### Role
- Safe environment for trying things out without affecting production VMs
- Disposable — can be reset to a clean snapshot at any time

### Notes
- Take a clean snapshot after initial setup so you can always roll back
- Not a critical VM — can be shut down when not in use to save resources
- Good candidate for testing Ansible playbooks, Docker configs, or new tools before applying to other VMs
