# Grafana Recovery

**Project:** DevOps Project  
**Trigger:** Cannot access Grafana, dashboards missing, or lost admin password

## Prerequisites
- SSH access to EC2 instance

---

## Grafana not loading on port 3000

### 1. Check the container is running
```bash
ssh -i ~/.ssh/your-key.pem ubuntu@ELASTIC_IP
docker compose ps grafana
```

### 2. If stopped, restart it
```bash
docker compose up -d grafana
```

### 3. Check logs for errors
```bash
docker compose logs grafana
```

---

## Reset admin password

If you have lost the Grafana admin password:

```bash
# Exec into the running Grafana container
docker compose exec grafana grafana-cli admin reset-admin-password newpassword123
```

Then log in at `http://ELASTIC_IP:3000` with `admin` / `newpassword123` and change it immediately.

---

## Prometheus data source showing as disconnected

1. Open Grafana → Configuration → Data Sources → Prometheus
2. Verify the URL is set to `http://prometheus:9090` (container name, not localhost)
3. Click Save & Test — should show green

> Using `localhost` here is a common mistake — inside Docker, services talk to each other by container/service name.

---

## Dashboards missing after restart

Dashboards are stored in the Grafana database. If you have not set up dashboard provisioning, they can be lost if the container volume is removed.

**To prevent this going forward**, export dashboards as JSON:

1. Open dashboard → Share → Export → Save to file
2. Commit the JSON files to your repo under `grafana/dashboards/`
3. Set up Grafana provisioning to auto-load them on startup

---

## Notes

- Grafana runs on port 3000, publicly accessible
- Default credentials: `admin` / `admin` — change on first login
- Consider restricting port 3000 in your AWS Security Group to your IP only
