# GeoStrap — Local Development & Support Runbook

**Version:** 0.1  
**Environment:** Local (Docker Compose)  
**Audience:** Level 1/2 Technical Support  
**Last updated:** March 2026

---

## Table of Contents

1. [What is GeoStrap?](#1-what-is-geostrap)
2. [System Overview](#2-system-overview)
3. [Prerequisites](#3-prerequisites)
4. [Starting the Application](#4-starting-the-application)
5. [Stopping the Application](#5-stopping-the-application)
6. [Loading Test Data](#6-loading-test-data)
7. [Running the RFID Simulator](#7-running-the-rfid-simulator)
8. [Using the Web Interface](#8-using-the-web-interface)
9. [API Quick Reference](#9-api-quick-reference)
10. [Common Issues & Fixes](#10-common-issues--fixes)
11. [Glossary](#11-glossary)

---

## 1. What is GeoStrap?

GeoStrap is an RFID-based asset tracking platform for logistics companies. It tracks physical shipping straps and load restraints fitted with RFID tags. The core use case is a truck driving through a gate fitted with an RFID reader — all tagged straps on the truck are scanned automatically, with no manual scanning required.

**Key concepts:**
- An **asset** is a physical strap or piece of load restraint equipment
- A **tag** is an RFID sticker attached to an asset
- A **reader** is the RFID scanning device installed at a gate or depot
- A **read** is a single scan event — a reader detecting a tag at a point in time

RFID is the primary tracking method. GPS is a premium add-on and may not be present in all environments.

---

## 2. System Overview

The local environment runs four components:

| Component | What it does | Where it runs |
|-----------|-------------|---------------|
| PostgreSQL 16 | Primary database — stores all assets, tags, reads, assignments | Docker container (`geostrap-postgres`) |
| Redis 7 | Rate limiting and session cache | Docker container (`geostrap-redis`) |
| Ingest API | FastAPI service — handles device data ingestion and all read/write operations | Local Python process (port 8000) |
| Web UI | React frontend — the browser-based admin interface | Local Node process (port 5173) |

```
Browser (5173) ──► Ingest API (8000) ──► PostgreSQL
                                    └──► Redis
ESP32 Devices ──► Ingest API (8000)
```

---

## 3. Prerequisites

The following must be installed on the machine before starting:

- **Docker Desktop** — must be running before any other step
- **Python 3.11+** — confirm with `python --version`
- **Node.js 18+** and **pnpm** — confirm with `node --version` and `pnpm --version`
- **Git Bash** (Windows) — all commands in this runbook are written for Git Bash

The repository must be cloned to `d:/GitHub/geostrap-web`.

---

## 4. Starting the Application

Follow these steps **in order**. Each step depends on the previous one completing successfully.

### Step 1 — Start Docker Desktop

Open Docker Desktop from the Start menu and wait until the whale icon in the system tray is steady (not animating). This usually takes 30–60 seconds.

### Step 2 — Open a terminal

Open Git Bash and navigate to the project root:

```bash
cd /d/GitHub/geostrap-web
```

### Step 3 — Start the database and cache

```bash
docker compose up -d
```

Verify both containers are running:

```bash
docker compose ps
```

Expected output — both should show `running`:

```
NAME               STATUS
geostrap-postgres  running
geostrap-redis     running
```

If you see a conflict error about container names already existing, run:

```bash
docker rm -f geostrap-redis geostrap-postgres
docker compose up -d
```

### Step 4 — Activate the Python virtual environment

```bash
source .venv/Scripts/activate
```

Your prompt will change to show `(.venv)` at the start. **This must be done every time you open a new terminal.**

### Step 5 — Set environment variables

```bash
export DATABASE_URL="postgresql+psycopg://geostrap:geostrap_dev_password@localhost:5432/geostrap"
export REDIS_URL="redis://localhost:6379"
export RFID_INGEST_SHARED_KEY="rfid_dev_shared_key"
```

These variables tell the API where to find the database and cache, and set the shared secret used to authenticate RFID reader devices.

### Step 6 — Run database migrations

Migrations create or update the database tables. This must be run from the `apps/api` directory:

```bash
cd apps/api
alembic upgrade head
cd ../..
```

Expected output ends with something like:

```
INFO  [alembic.runtime.migration] Running upgrade ... -> 20260308_000002
```

If it says `already at head`, the database is already up to date — this is fine, continue.

### Step 7 — Load seed data

Seed scripts populate the database with test organisations, assets, readers, and tags. Only run these **once** on a fresh database. Running them again on an existing database will cause duplicate key errors — this is harmless but can be ignored.

```bash
python -m apps.ingest.scripts.seed_dev
python -m apps.ingest.scripts.seed_rfid_dev
```

### Step 8 — Start the API

```bash
uvicorn apps.ingest.main:app --host 0.0.0.0 --port 8000
```

Leave this terminal open — the API runs in the foreground. You will see request logs here as the application is used.

Confirm it is running by opening a **second terminal** and running:

```bash
curl http://localhost:8000/healthz
```

Expected response: `{"status":"ok"}`

### Step 9 — Start the web interface

In the second terminal, navigate to the project and start the frontend:

```bash
cd /d/GitHub/geostrap-web
pnpm web:dev
```

The web interface will be available at: **http://localhost:5173**

---

## 5. Stopping the Application

### Stop the web interface
In the terminal running `pnpm web:dev`, press `Ctrl+C`.

### Stop the API
In the terminal running `uvicorn`, press `Ctrl+C`.

### Stop the database and cache

```bash
docker compose down
```

This stops the containers but preserves all data. To stop **and delete all data** (full reset):

```bash
docker compose down -v
```

> **Warning:** `down -v` permanently deletes the database. You will need to re-run migrations and seed scripts after this.

---

## 6. Loading Test Data

The seed scripts create the following test data:

### What `seed_dev` creates

| Type | Details |
|------|---------|
| Organisation | ID: 1, Name: Dev Org |
| Asset | ID: 1, Code: `strap-123`, Name: Strap #123 |
| Device | A GPS device linked to asset 1 |
| User | A dev admin user |

### What `seed_rfid_dev` creates

| Type | Details |
|------|---------|
| Asset | ID: 2, Code: `strap-rfid-dev-001`, Name: RFID Dev Strap #1 |
| Reader | ID: 1, UID: `reader-dev-001`, Name: Dev RFID Reader 001 |
| Tags | Tag 1 (EPC: `3034AABBCCDDEEFF00112233`) assigned to Asset 2 |
|      | Tag 2 (EPC: `3034AABBCCDDEEFF00112234`) unassigned |

---

## 7. Running the RFID Simulator

The simulator generates fake RFID scan data and sends it to the API. This is used for testing and demonstration without physical hardware.

The API must be running before starting the simulator.

### Burst mode — single scan of all tags

Sends one batch containing all configured tags, then exits. Use this to quickly populate the event timeline.

```bash
python -m apps.ingest.scripts.simulate_rfid --mode burst
```

### Continuous mode — ongoing scanning

Sends a batch of 3–8 random tags every 5 seconds, simulating a reader at a gate. Press `Ctrl+C` to stop.

```bash
python -m apps.ingest.scripts.simulate_rfid --mode continuous
```

You will see a summary line printed for each batch:

```
[14:32:01] Sent batch: 5 tags | EPCs: 3034AA..., 3034AA..., ...
```

After running the simulator, refresh the web interface to see the new scan events.

---

## 8. Using the Web Interface

Open **http://localhost:5173** in a browser. The application has three main pages accessible from the navigation bar.

---

### 8.1 Assets Page (`/assets`)

Lists all tracked assets in the organisation.

| Column | What it means |
|--------|--------------|
| **ID** | Unique database identifier for the asset. Use this when looking up records or reporting issues. |
| **Asset** | The asset's display name and code. Click to open the asset detail page. The code (in brackets) is the short reference used in operations. |
| **Assigned Tag EPC** | The EPC (Electronic Product Code) of the RFID tag currently assigned to this asset. `-` means no tag is assigned. |
| **Last GPS** | Timestamp of the most recent GPS position recorded for this asset. `-` means no GPS data has been received. |
| **Last RFID** | Timestamp of the most recent RFID scan of this asset's assigned tag. `-` means the tag has never been scanned. |
| **Status** | A combined summary: GPS active/No GPS, and RFID presence status (present/unknown). |

---

### 8.2 Asset Detail Page (`/assets/:id`)

Shows full detail for a single asset including its event history.

#### Status badges (top of page)

| Badge | Colour | What it means |
|-------|--------|--------------|
| **GPS ACTIVE** | Green | The asset has a GPS device and has reported a position |
| **NO GPS** | Gray | No GPS device is linked to this asset |
| **PRESENT** | Green | The asset's tag was seen by a reader recently |
| **UNKNOWN** | Gray | The tag has never been scanned or status is undetermined |
| **EPC code** | Blue | The RFID tag currently assigned to this asset |
| **NO TAG** | Gray | No RFID tag is assigned to this asset |

#### RFID Snapshot card

| Field | What it means |
|-------|--------------|
| **Last seen** | When the tag was most recently detected by any reader |
| **Reader** | The name of the reader that last detected the tag |
| **RSSI** | Signal strength of the last scan. Strong (above -55 dBm), Moderate (-55 to -70 dBm), Weak (below -70 dBm). A weaker signal may indicate the tag was at the edge of read range or obscured. |
| **Reader UID** | The unique identifier of the reader hardware |

#### GPS Snapshot card

| Field | What it means |
|-------|--------------|
| **Last position** | Timestamp of the most recent GPS fix |
| **Lat / Lon** | Geographic coordinates of the last known position |
| **Battery** | Battery percentage of the GPS device at last report |

#### Event Timeline

A chronological history of all events for this asset, newest first. Three event types appear:

| Indicator colour | Event type | What it means |
|-----------------|-----------|--------------|
| **Teal dot** | RFID scan | The asset's tag was detected by a reader. Shows reader name, RSSI signal strength, and antenna port. |
| **Blue dot** | GPS position | A GPS position was recorded for the asset. Shows coordinates. |
| **Amber dot** | Tag assignment | A tag was assigned to or unassigned from this asset. Shows the tag EPC and the action taken. |

The timeline is the primary tool for investigating an asset's history — for example, confirming a strap was scanned at a gate at a particular time.

---

### 8.3 RFID Reads Page (`/rfid/reads`)

A live feed of all recent RFID scan events across all readers, newest first.

| Column | What it means |
|--------|--------------|
| **Time** | When the scan occurred |
| **Reader** | Name and UID of the reader that detected the tag |
| **EPC** | The Electronic Product Code of the tag that was scanned |
| **Known tag** | `known` — the EPC matches a registered tag in the system. `unknown` — the EPC is not registered (the tag exists physically but has not been added to GeoStrap). |
| **Assigned asset** | The asset ID the tag is currently assigned to. `-` means the tag is not assigned to any asset. |
| **RSSI** | Signal strength in dBm. More negative = weaker signal. Typical gate read range: -45 to -70. |
| **Antenna** | Which antenna port on the reader detected the tag (readers can have multiple antennas) |
| **Direction** | Travel direction if configured on the reader (e.g. entry/exit). `-` means direction detection is not configured. |

Use the **Limit** dropdown to control how many reads are shown (25 / 50 / 100 / 200).

---

### 8.4 Tag Assignments Page (`/rfid/tags`)

Shows all RFID tags and their current assignment status. Allows assigning tags to assets and unassigning them.

| Column | What it means |
|--------|--------------|
| **Tag ID** | Unique database identifier for the tag |
| **EPC** | The tag's Electronic Product Code — the unique identifier encoded on the tag chip |
| **Status** | `active` — tag is in use. Other values may indicate a retired or lost tag. |
| **Assigned asset** | The asset this tag is currently assigned to, or `—` if unassigned |
| **Action** | Controls to assign or unassign the tag |

#### Assigning a tag

1. Find a tag with no assigned asset (shows `—` in the Assigned Asset column)
2. Use the **Select asset** dropdown to choose the target asset
3. Click **Assign**
4. The table will refresh and show the new assignment

#### Unassigning a tag

1. Find a tag that is currently assigned (shows the asset name in the Assigned Asset column)
2. Click **Unassign**
3. The table will refresh and the tag will show as unassigned

> **Note:** A tag can only be assigned to one asset at a time. Assigning a tag that is already assigned to asset A to a different asset B will automatically close the previous assignment and create a new one. The full assignment history is preserved in the database and visible in the asset's event timeline.

---

## 9. API Quick Reference

The API runs at `http://localhost:8000`. All read endpoints require `?organization_id=1`.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/healthz` | Health check — returns `{"status":"ok"}` if running |
| GET | `/assets?organization_id=1` | List all assets with current state |
| GET | `/assets/{id}?organization_id=1` | Single asset detail |
| GET | `/assets/{id}/timeline?organization_id=1` | Asset event timeline |
| GET | `/rfid/reads?organization_id=1&limit=100` | Recent RFID scan events |
| GET | `/rfid/tags?organization_id=1` | All RFID tags and assignment status |
| GET | `/rfid/readers?organization_id=1` | All registered RFID readers |
| POST | `/rfid/tags/{id}/assign?organization_id=1` | Assign a tag to an asset |
| POST | `/rfid/tags/{id}/unassign?organization_id=1` | Unassign a tag |
| POST | `/ingest/v1/rfid-reads/batch` | Ingest a batch of RFID reads (device auth required) |

Full interactive API documentation is available at: **http://localhost:8000/docs**

---

## 10. Common Issues & Fixes

### "command not found: alembic" or "command not found: uvicorn"

The Python virtual environment is not active. Run:

```bash
source .venv/Scripts/activate
```

Then retry the command.

### "connection refused" on port 8000

The API is not running. Start it with:

```bash
uvicorn apps.ingest.main:app --host 0.0.0.0 --port 8000
```

### "connection refused" or database errors in the API

The PostgreSQL container is not running. Check:

```bash
docker compose ps
```

If `geostrap-postgres` is not running:

```bash
docker compose up -d
```

### Container name conflict on `docker compose up`

A leftover container from a previous session is blocking startup:

```bash
docker rm -f geostrap-redis geostrap-postgres
docker compose up -d
```

### Alembic error: "Path doesn't exist: alembic"

You are running the alembic command from the wrong directory. Always run it from `apps/api`:

```bash
cd apps/api
alembic upgrade head
cd ../..
```

### The web interface shows no data

1. Confirm the API is running: `curl http://localhost:8000/healthz`
2. Confirm seed data has been loaded (see [Section 6](#6-loading-test-data))
3. Check the browser console for errors (F12 → Console tab)

### Reads are appearing in the API but not updating in the browser

The web interface does not auto-refresh. Manually refresh the browser page after running the simulator or after scanning physical tags.

---

## 11. Glossary

| Term | Definition |
|------|-----------|
| **Asset** | A physical item being tracked — in GeoStrap, typically a shipping strap or load restraint |
| **EPC** | Electronic Product Code — the unique identifier stored on an RFID tag chip, typically a hex string like `3034AABBCCDDEEFF00112233` |
| **Reader** | An RFID scanning device installed at a fixed location such as a gate or depot entrance |
| **Tag** | A small RFID sticker or label attached to an asset containing an EPC |
| **Read** | A single detection event — a reader scanning a tag at a specific point in time |
| **RSSI** | Received Signal Strength Indicator — measures how strongly a tag's signal was received, in dBm. Less negative = stronger signal. Typical range: -40 (very strong) to -80 (weak). |
| **Antenna port** | RFID readers can have multiple antennas. The port number identifies which antenna detected the tag. |
| **Presence status** | Whether a tag is currently considered present (recently scanned) or unknown (not seen recently) |
| **Assignment** | The link between a tag and an asset. One tag can only be assigned to one asset at a time. History of all assignments is preserved. |
| **Organization** | The top-level tenant in GeoStrap. All data is scoped to an organisation. In the dev environment, Organisation ID 1 is used. |
| **Ingest** | The process of receiving data from RFID readers and GPS devices and storing it in the database |
| **Seed data** | Pre-generated test data loaded into the database for development and testing purposes |
| **Migration** | A versioned database schema change. Alembic migrations ensure the database structure matches what the application expects. |
