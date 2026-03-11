# Runbooks

Operational procedures for deployments, recovery, and common tasks.

---

## Available Runbooks

| Runbook | Project | Trigger |
|---|---|---|
| [EC2 Instance Recovery](ec2-recovery.md) | DevOps Project | Instance unreachable or stopped |
| [Deploy / Restart Stack](docker-compose-deploy.md) | DevOps Project | Deploy changes or restart after failure |
| [Grafana Recovery](grafana-recovery.md) | DevOps Project | Grafana down or password lost |
| [VM Snapshot & Restore](vm-snapshot.md) | Home Server | Before major changes or disaster recovery |
| [Immich Backup Restore](immich-restore.md) | Home Server | Photo data loss or VM failure |

---

## Runbook Template

Use this when adding a new runbook:

```markdown
# Runbook Title

**Project:** Project name  
**Trigger:** When to use this runbook

## Prerequisites
- Access or tools needed

## Steps

### 1. Step name
description and commands

## Rollback
How to undo if something goes wrong

## Notes
Anything worth knowing
```
