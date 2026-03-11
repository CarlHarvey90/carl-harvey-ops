# Deploy / Restart Stack

**Project:** DevOps Project  
**Trigger:** Deploying changes, restarting after failure, or updating container images

## Prerequisites
- SSH access to EC2 instance
- Docker and Docker Compose installed on instance

## SSH into the instance

```bash
ssh -i ~/.ssh/your-key.pem ubuntu@ELASTIC_IP
```

## Check current stack status

```bash
cd ~/your-project-folder
docker compose ps
```

All services should show `Up`. If any show `Exit` or `Restarting`, see troubleshooting below.

## Deploy updated code

```bash
# Pull latest code if using git on the server
git pull origin main

# Rebuild and restart only changed containers
docker compose up -d --build
```

## Restart the full stack

```bash
# Stop all containers
docker compose down

# Start fresh
docker compose up -d
```

## View logs

```bash
# All services
docker compose logs -f

# Single service
docker compose logs -f web
docker compose logs -f prometheus
docker compose logs -f grafana
```

## Troubleshooting

### Container keeps restarting
```bash
# Check logs for the failing container
docker compose logs web

# Check resource usage
docker stats
```

### Flask app not responding on port 5000
```bash
# Verify container is running
docker compose ps web

# Check the port is bound
sudo ss -tlnp | grep 5000
```

### Prometheus not scraping
```bash
# Open Prometheus targets page
# http://ELASTIC_IP:9090/targets
# Look for any targets showing DOWN state
```

## Rollback

If a deployment breaks the stack:

```bash
# Revert code to previous commit
git revert HEAD

# Rebuild with previous version
docker compose up -d --build
```

## Notes

- PostgreSQL data persists via Docker volume — `docker compose down` does NOT delete data
- To wipe the database: `docker compose down -v` (destructive — use with caution)
- Free tier EC2 has limited RAM — avoid running `--build` during high traffic
