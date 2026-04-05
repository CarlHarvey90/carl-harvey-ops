# GeoStrap — Local Development & Support Runbook

**Version:** 0.3  
**Environment:** Local (Docker Compose)  
**Audience:** Level 1/2 Technical Support  
**Last updated:** April 2026

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
10. [Observability](#10-observability)
11. [Common Issues & Fixes](#11-common-issues--fixes)
12. [Glossary](#12-glossary)

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

The local environment runs five components across three terminals:

| Component | What it does | Where it runs |
|-----------|-------------|---------------|
| PostgreSQL 16 | Primary database — stores all assets, tags, reads, assignments, audit logs | Docker container (`geostrap-postgres`) |
| Redis 7 | Rate limiting for ingest service | Docker container (`geostrap-redis`) |
| Ingest API | FastAPI service — device-facing, handles RFID/GPS data ingestion | Local Python process (port 8000) |
| API | FastAPI service — human-facing, handles auth, admin, RFID management, audit logs | Local Python process (port 8001) |
| Web UI | React frontend — the browser-based admin interface | Local Node process (port 5173) |

```
Browser (5173) ──► API (8001) ──► PostgreSQL
                                    
ESP32 Devices ──► Ingest (8000) ──► PostgreSQL
                               └──► Redis (rate limiting)
```

Both backend services share code via `apps/shared/` (database functions, schemas, auth utilities, metrics).

All data is **organisation-scoped** — queries filter by the organisation derived from the authenticated user's JWT token. There is no `organization_id` query parameter.

---

## 3. Prerequisites

The following must be installed on the machine before starting:

- **Docker** (Engine + Compose, or Docker Desktop) — must be running before any other step
- **Python 3.11+** — confirm with `python --version`
- **Node.js 18+** and **pnpm** — confirm with `node --version` and `pnpm --version`
- **Git** — for cloning the repository

The repository must be cloned locally. See `.env.example` for required environment variables.

---

## 4. Starting the Application

Follow these steps **in order**. Each step depends on the previous one completing successfully.

### Step 1 — Start Docker

Ensure Docker is running. On Linux:

```bash
sudo systemctl start docker
```

On Windows/Mac, open Docker Desktop and wait for it to be ready.

### Step 2 — Open a terminal

Navigate to the project root:

```bash
cd <REPO_PATH>
```

### Step 3 — Start the database and cache

```bash
docker compose up -d postgres redis
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

### Step 4 — Activate the Python virtual environment

```bash
source <VENV_PATH>/bin/activate
```

Your prompt will change to show the venv name. **This must be done every time you open a new terminal.**

### Step 5 — Set environment variables

Set the following in your terminal session:

| Variable | Purpose | Example |
|----------|---------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql+psycopg://<DB_USER>:<DB_PASSWORD>@localhost:5432/<DB_NAME>` |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379` |
| `RFID_INGEST_SHARED_KEY` | Shared secret for device auth | `<INGEST_SHARED_KEY>` |
| `JWT_SECRET` | Secret for signing user JWT tokens | `<JWT_SECRET>` |

Not all services need every variable:

| Variable | Ingest (8000) | API (8001) | Seed scripts |
|----------|---------------|------------|--------------|
| `DATABASE_URL` | Required | Required | Required |
| `REDIS_URL` | Required | — | — |
| `RFID_INGEST_SHARED_KEY` | Required | — | — |
| `JWT_SECRET` | — | Required | — |

### Step 6 — Run database migrations

Migrations create or update the database tables. This must be run from the `apps/api` directory:

```bash
cd apps/api
alembic upgrade head
cd ../..
```

If it says `already at head`, the database is already up to date — this is fine, continue.

### Step 7 — Load seed data

Seed scripts populate the database with test organisations, assets, readers, and tags. Only run these **once** on a fresh database. Running them again on an existing database will cause duplicate key errors — this is harmless and can be ignored.

```bash
python -m apps.ingest.scripts.seed_dev
python -m apps.ingest.scripts.seed_rfid_dev
```

`seed_dev` creates a login user with the `admin` role. Credentials are printed to the console.

### Step 8 — Start backend services

Two services run in **separate terminals**, both from the repo root with venv active and env vars set.

**Terminal A — Ingest service (device-facing, port 8000):**

```bash
uvicorn apps.ingest.main:app --host 0.0.0.0 --port 8000 --reload
```

Requires: `DATABASE_URL`, `REDIS_URL`, `RFID_INGEST_SHARED_KEY`.

Confirm it is running:

```bash
curl http://localhost:8000/healthz
```

Expected response: `{"status":"ok"}`

**Terminal B — API service (human-facing, port 8001):**

```bash
uvicorn apps.api.main:app --host 0.0.0.0 --port 8001 --reload
```

Requires: `DATABASE_URL`, `JWT_SECRET`.

Confirm it is running:

```bash
curl http://localhost:8001/healthz
```

### Step 9 — Start the web interface

In a third terminal, navigate to the project and start the frontend:

```bash
cd <REPO_PATH>
pnpm web:dev
```

The web interface will be available at: **http://localhost:5173**

You will be redirected to `/login` — use the dev credentials from step 7.

Confirm `apps/web/.env` exists with:

```
VITE_API_BASE_URL=http://localhost:8001
```

---

## 5. Stopping the Application

### Stop the web interface
In the terminal running `pnpm web:dev`, press `Ctrl+C`.

### Stop the backend services
In each terminal running `uvicorn`, press `Ctrl+C`.

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
| Organisation | Dev organisation |
| Asset | A test asset |
| Device | A GPS device linked to the test asset |
| User | An admin user (credentials printed to console) |

### What `seed_rfid_dev` creates

| Type | Details |
|------|---------|
| Asset | An RFID-enabled test asset |
| Reader | A dev RFID reader |
| Tags | Two tags — one assigned to the asset, one unassigned |

---

## 7. Running the RFID Simulator

The simulator generates fake RFID scan data and sends it to the ingest API. This is used for testing and demonstration without physical hardware.

The ingest API must be running before starting the simulator.

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

Open **http://localhost:5173** in a browser. You will be redirected to the login page. After logging in, the application has several pages accessible from the navigation bar.

Admin users see additional nav links for **Users** (user management) and **Audit log**.

---

### 8.1 Assets Page (`/assets`)

Lists all tracked assets in the organisation.

| Column | What it means |
|--------|--------------|
| **ID** | Unique database identifier for the asset. |
| **Asset** | The asset's display name and code. Click to open the asset detail page. |
| **Assigned Tag EPC** | The EPC of the RFID tag currently assigned to this asset. `—` means no tag is assigned. |
| **Last GPS** | Timestamp of the most recent GPS position recorded for this asset. |
| **Last RFID** | Timestamp of the most recent RFID scan of this asset's assigned tag. |
| **Status** | GPS active/No GPS, and RFID presence status (present/unknown). |

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
| **RSSI** | Signal strength of the last scan. Strong (above -55 dBm), Moderate (-55 to -70 dBm), Weak (below -70 dBm). |
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
| **Teal dot** | RFID scan | The asset's tag was detected by a reader. Shows reader name, RSSI, and antenna port. |
| **Blue dot** | GPS position | A GPS position was recorded. Shows coordinates. |
| **Amber dot** | Tag assignment | A tag was assigned to or unassigned from this asset. Shows the tag EPC and the action taken. |

---

### 8.3 RFID Reads Page (`/rfid/reads`)

A live feed of all recent RFID scan events across all readers, newest first.

| Column | What it means |
|--------|--------------|
| **Time** | When the scan occurred |
| **Reader** | Name and UID of the reader that detected the tag |
| **EPC** | The Electronic Product Code of the tag that was scanned |
| **Known tag** | `known` — the EPC matches a registered tag. `unknown` — the EPC is not registered. |
| **Assigned asset** | The asset ID the tag is currently assigned to. `—` means unassigned. |
| **RSSI** | Signal strength in dBm. More negative = weaker signal. |
| **Antenna** | Which antenna port on the reader detected the tag |
| **Direction** | Travel direction if configured (e.g. entry/exit). `—` if not configured. |

Use the **Limit** dropdown to control how many reads are shown (25 / 50 / 100 / 200).

---

### 8.4 Tag Assignments Page (`/rfid/tags`)

Shows all RFID tags and their current assignment status. Allows assigning tags to assets and unassigning them.

| Column | What it means |
|--------|--------------|
| **Tag ID** | Unique database identifier for the tag |
| **EPC** | The tag's Electronic Product Code |
| **Status** | `active` — tag is in use. Other values may indicate a retired or lost tag. |
| **Assigned asset** | The asset this tag is currently assigned to, or `—` if unassigned |
| **Action** | Controls to assign or unassign the tag |

#### Assigning a tag

1. Find a tag with no assigned asset
2. Use the **Select asset** dropdown to choose the target asset
3. Click **Assign**
4. The table will refresh and show the new assignment

#### Unassigning a tag

1. Find a tag that is currently assigned
2. Click **Unassign**
3. The table will refresh and the tag will show as unassigned

> **Note:** A tag can only be assigned to one asset at a time. Assigning a tag that is already assigned will automatically close the previous assignment. The full assignment history is preserved and visible in the asset's event timeline.

---

### 8.5 User Management Page (`/admin/users`) — Admin only

Allows admin users to create, edit, and deactivate user accounts.

| Column | What it means |
|--------|--------------|
| **Email** | The user's login email |
| **Role** | `admin` or `member` — admins can manage users and view audit logs |
| **Active** | Whether the user can log in |
| **Actions** | Edit role/status, reset password, deactivate |

Only users with the `admin` role can access this page. Non-admins are redirected to the home page.

---

### 8.6 Audit Log Page (`/admin/audit`) — Admin only

A read-only log of all significant user actions across the platform (logins, tag assignments, user changes, etc.).

| Column | What it means |
|--------|--------------|
| **Timestamp** | When the action occurred |
| **User** | The email of the user who performed the action |
| **Action** | The type of action (e.g. `LOGIN_SUCCESS`, `TAG_ASSIGNED`, `USER_CREATED`) |
| **Resource** | The type of object affected (e.g. `rfid_tag`, `user`) |
| **Resource ID** | The ID of the affected object |
| **Detail** | A human-readable summary of what changed |
| **IP** | The IP address the action was performed from |

Filters are available for user, action type, and date range. The log auto-refreshes every 30 seconds.

Action badges are colour-coded:

| Colour | Action types |
|--------|-------------|
| **Blue** | Auth actions — `LOGIN_SUCCESS`, `LOGOUT`, `TOKEN_REFRESHED` |
| **Teal** | Write actions — `*_CREATED`, `*_UPDATED`, `*_ASSIGNED` |
| **Amber** | Destructive actions — `LOGIN_FAILED`, `*_DEACTIVATED`, `*_UNASSIGNED` |

The audit log is append-only — entries are never modified or deleted.

---

## 9. API Quick Reference

Two backend services handle different concerns:

### Ingest API — `http://localhost:8000`

Device-facing endpoints. Authenticated via `X-DEVICE-KEY` header.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/healthz` | Health check |
| GET | `/metrics` | Prometheus metrics (unauthenticated) |
| POST | `/ingest/v1/position` | Ingest a GPS position |
| POST | `/ingest/v1/rfid-read` | Ingest a single RFID read |
| POST | `/ingest/v1/rfid-reads/batch` | Ingest a batch of RFID reads |

### API — `http://localhost:8001`

Human-facing endpoints. Authenticated via JWT Bearer token (except `/auth/login` and `/healthz`).

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/healthz` | Health check |
| GET | `/metrics` | Prometheus metrics (unauthenticated) |
| POST | `/auth/login` | Login — returns access + refresh tokens |
| POST | `/auth/refresh` | Refresh an access token |
| POST | `/auth/logout` | Logout |
| GET | `/assets` | List all assets with current state |
| GET | `/assets/{id}` | Single asset detail |
| GET | `/assets/{id}/timeline` | Asset event timeline |
| GET | `/rfid/readers` | All registered RFID readers |
| GET | `/rfid/tags` | All RFID tags and assignment status |
| GET | `/rfid/reads?limit=100` | Recent RFID scan events |
| POST | `/rfid/readers` | Create a reader |
| PUT | `/rfid/readers/{id}` | Update a reader |
| POST | `/rfid/tags` | Create a tag |
| PUT | `/rfid/tags/{id}` | Update a tag |
| POST | `/rfid/tags/{id}/assign` | Assign a tag to an asset |
| POST | `/rfid/tags/{id}/unassign` | Unassign a tag |
| GET | `/admin/users` | List users (admin only) |
| POST | `/admin/users` | Create user (admin only) |
| PATCH | `/admin/users/{id}` | Update user (admin only) |
| POST | `/admin/users/{id}/password` | Reset user password (admin only) |
| DELETE | `/admin/users/{id}` | Deactivate user (admin only) |
| GET | `/admin/audit` | Query audit logs (admin only) |

Interactive API documentation: **http://localhost:8001/docs**

---

## 10. Observability

Both services expose Prometheus-compatible metrics at `/metrics` (unauthenticated).

### Auto-instrumented metrics

Provided by `prometheus-fastapi-instrumentator`:

- `http_requests_total` — request count by handler, method, and status code
- `http_request_duration_seconds` — request latency histogram

`/healthz` traffic is excluded from these metrics.

### Custom business metrics

| Metric | Type | Service | Description |
|--------|------|---------|-------------|
| `geostrap_rfid_reads_total` | Counter | Ingest | Total RFID reads received, labelled by `reader_id` and `status` |
| `geostrap_rfid_batches_total` | Counter | Ingest | Total batch POST requests, labelled by `status` |
| `geostrap_gps_positions_total` | Counter | Ingest | Total GPS positions received, labelled by `status` |
| `geostrap_active_rfid_readers` | Gauge | Ingest | Readers that posted within the last 5 minutes (updated every 60s) |
| `geostrap_api_auth_attempts_total` | Counter | API | Login attempts, labelled by `status` (success/failed) |

These endpoints are ready for scraping when a Prometheus server is deployed.

---

## 11. Common Issues & Fixes

### "command not found: alembic" or "command not found: uvicorn"

The Python virtual environment is not active. Activate it then retry.

### "connection refused" on port 8000 or 8001

The corresponding backend service is not running. Start it with the appropriate `uvicorn` command (see Step 8).

### "connection refused" or database errors in the API

The PostgreSQL container is not running. Check:

```bash
docker compose ps
```

If `geostrap-postgres` is not running:

```bash
docker compose up -d postgres redis
```

### Container name conflict on `docker compose up`

A leftover container from a previous session is blocking startup:

```bash
docker rm -f geostrap-redis geostrap-postgres
docker compose up -d postgres redis
```

### Alembic error: "Path doesn't exist: alembic"

You are running the alembic command from the wrong directory. Always run it from `apps/api`:

```bash
cd apps/api
alembic upgrade head
cd ../..
```

### `DATABASE_URL is required`

Environment variables are not set in the current terminal session. Export them before running any Python command.

### `JWT_SECRET is required`

The `JWT_SECRET` env var is not set in the terminal running the API service (port 8001). The ingest service does not need this variable.

### `relation "audit_logs" does not exist`

Alembic migration not run after audit logging was added. Run `cd apps/api && alembic upgrade head`.

### The web interface shows no data

1. Confirm the API service is running: `curl http://localhost:8001/healthz`
2. Confirm seed data has been loaded (see [Section 6](#6-loading-test-data))
3. Check `apps/web/.env` has `VITE_API_BASE_URL=http://localhost:8001`
4. Check the browser console for errors (F12 → Console tab)

### Reads are appearing in the API but not updating in the browser

The reads page does not auto-refresh. Manually refresh the browser page after running the simulator. The audit log page auto-refreshes every 30 seconds.

---

## 12. Glossary

| Term | Definition |
|------|-----------|
| **Asset** | A physical item being tracked — in GeoStrap, typically a shipping strap or load restraint |
| **EPC** | Electronic Product Code — the unique identifier stored on an RFID tag chip, typically a hex string |
| **Reader** | An RFID scanning device installed at a fixed location such as a gate or depot entrance |
| **Tag** | A small RFID sticker or label attached to an asset containing an EPC |
| **Read** | A single detection event — a reader scanning a tag at a specific point in time |
| **RSSI** | Received Signal Strength Indicator — measures how strongly a tag's signal was received, in dBm. Less negative = stronger signal. |
| **Antenna port** | RFID readers can have multiple antennas. The port number identifies which antenna detected the tag. |
| **Presence status** | Whether a tag is currently considered present (recently scanned) or unknown (not seen recently) |
| **Assignment** | The link between a tag and an asset. One tag can only be assigned to one asset at a time. History is preserved. |
| **Organization** | The top-level tenant in GeoStrap. All data is scoped to an organisation. |
| **Ingest** | The process of receiving data from RFID readers and GPS devices and storing it in the database |
| **API** | The human-facing backend service that serves the web UI and admin functionality |
| **JWT** | JSON Web Token — used to authenticate human users. Issued on login, passed as a Bearer token in API requests. |
| **Audit log** | An append-only record of all significant user actions in the system |
| **Seed data** | Pre-generated test data loaded into the database for development and testing purposes |
| **Migration** | A versioned database schema change. Alembic migrations ensure the database structure matches what the application expects. |
| **Prometheus** | An open-source monitoring toolkit. Both services expose `/metrics` endpoints for scraping. |
