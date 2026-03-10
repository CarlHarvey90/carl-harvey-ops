# EC2 Instance Recovery

**Project:** DevOps Project  
**Trigger:** EC2 instance unreachable or stopped unexpectedly

## Prerequisites
- AWS CLI configured (`aws configure`)
- SSH key for instance

## Steps

### 1. Check instance state
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=devops-project" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' \
  --output table
```

### 2. Start stopped instance
```bash
aws ec2 start-instances --instance-ids <INSTANCE_ID>
```

### 3. Wait for running state
```bash
aws ec2 wait instance-running --instance-ids <INSTANCE_ID>
```

### 4. SSH in and verify services
```bash
ssh -i ~/.ssh/your-key.pem ec2-user@<PUBLIC_IP>
sudo systemctl status <your-service>
```

## Rollback
If instance is corrupted, restore from latest AMI snapshot.

## Notes
- Free tier: ensure instance stays t2.micro to avoid charges
