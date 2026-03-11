# Carl Harvey — Ops Runbooks

Personal infrastructure documentation, runbooks, and architecture notes.

---

## Projects at a Glance

| Project | Status | Infrastructure | Domain | Monthly Cost |
|---|---|---|---|---|
| DevOps Project | Active | AWS EC2 Free Tier | carlharvey.dev | ~£0.50 |
| Home Server | Active | Physical / VMs | local | £0 |
| DevOps Portfolio | Planning | TBD | TBD | — |
| Geostrap | Planning | TBD | TBD | — |

---

## DevOps Project — Stack Summary

Flask API + PostgreSQL + Prometheus + Grafana running on EC2 via Docker Compose.

| Service | Port | Public |
|---|---|---|
| Flask App | 5000 | Yes |
| Grafana | 3000 | Yes |
| Prometheus | 9090 | Yes |
| PostgreSQL | 5432 | No |

---

## Quick Links

- [DevOps Project](projects/devops-project/index.md)
- [Home Server](projects/home-server/index.md)
- [Architecture Diagrams](architecture/index.md)
- [Runbooks](runbooks/index.md)
- [Incident Log](incidents/index.md)
