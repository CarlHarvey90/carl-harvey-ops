# GeoStrap

**Status:** 🟢 Active Development

## Overview

GeoStrap is an RFID/GPS asset tracking platform for logistics companies. It tracks physical shipping straps and load restraints fitted with RFID tags. The primary use case is automated scanning — a truck drives through a gate fitted with an RFID reader and all tagged assets are detected without manual intervention.

## Architecture

A pnpm monorepo with two language stacks:

| Component | Stack | Port | Purpose |
|-----------|-------|------|---------|
| Ingest service | Python / FastAPI | 8000 | Device-facing — receives GPS and RFID data from hardware |
| API service | Python / FastAPI | 8001 | Human-facing — auth, admin, RFID management, audit logging |
| Web UI | React 18 / TypeScript / Vite | 5173 | Admin frontend |
| Shared library | Python | — | Common code: DB, schemas, auth, metrics |

Both backend services share code via `apps/shared/` and write to the same PostgreSQL database. Redis is used by both services for rate limiting — the ingest service for per-device RFID throttling and the API service for login brute force protection.

**Auth:** JWT-based for human users (API service), shared key header for devices (ingest service). All data is organisation-scoped via the authenticated user's JWT token.

**Observability:** Both services expose Prometheus `/metrics` endpoints with auto-instrumented HTTP metrics and custom business counters.

**Audit:** All significant user actions are logged to an append-only `audit_logs` table, viewable by admins in the web UI.

## Infrastructure Requirements

| Service | Local Dev | Production |
|---------|-----------|------------|
| PostgreSQL 16 | Docker Compose | TBD |
| Redis 7 | Docker Compose | TBD |
| Prometheus | Not deployed (endpoints ready) | TBD |

## Runbooks

- [Local Development Runbook](geostrap-local-runbook.md)
