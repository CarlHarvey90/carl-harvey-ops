# EC2 Instance Recovery

**Project:** DevOps Project  
**Trigger:** EC2 instance unreachable, stopped unexpectedly, or Docker stack down after reboot

## Prerequisites
- AWS CLI configured (`aws configure`) or access to AWS Console
- SSH key for the instance

---

## Check instance state

### Via AWS CLI
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=devops-project" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' \
  --output table
```

### Via AWS Console
EC2 → Instances → check State column

---

## Instance is stopped — start it

```bash
aws ec2 start-instances --instance-ids YOUR_INSTANCE_ID

# Wait until running
aws ec2 wait instance-running --instance-ids YOUR_INSTANCE_ID
```

> Note: Elastic IP stays attached, so your IP will not change on restart.

---

## Instance is running but services are down

After a reboot, Docker containers do not always restart automatically unless configured to do so.

```bash
# SSH in
ssh -i ~/.ssh/your-key.pem ubuntu@ELASTIC_IP

# Check Docker is running
sudo systemctl status docker

# Check stack status
cd ~/your-project-folder
docker compose ps

# Start the stack if it is down
docker compose up -d
```

---

## Make stack start automatically on reboot

To avoid manual restarts after future reboots, ensure containers have a restart policy in your `docker-compose.yml`:

```yaml
services:
  web:
    restart: unless-stopped
  db:
    restart: unless-stopped
  prometheus:
    restart: unless-stopped
  grafana:
    restart: unless-stopped
```

Then apply it:
```bash
docker compose up -d
```

---

## Instance is unreachable via SSH

1. Check the instance is in `running` state (see above)
2. Check the Elastic IP is still associated in the AWS Console
3. Check Security Group allows port 22 from your IP
4. Try connecting with verbose output:
```bash
ssh -v -i ~/.ssh/your-key.pem ubuntu@ELASTIC_IP
```

---

## Last resort — restore from snapshot

If the instance is corrupted and unrecoverable:

1. AWS Console → EC2 → Snapshots → find latest snapshot
2. Actions → Create Image from Snapshot
3. Launch new instance from that AMI
4. Re-attach the Elastic IP to the new instance

---

## Notes

- t2.micro has 1GB RAM — monitor memory usage with `free -h` after recovery
- If free tier hours are exhausted, the instance will incur charges — check AWS Billing dashboard
- Always verify `docker compose ps` shows all 4 services as `Up` after recovery
