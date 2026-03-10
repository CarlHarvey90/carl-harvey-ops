# AWS Infrastructure

## Account Overview
- **Region:** eu-west-1
- **Free Tier Active:** Yes

## Resource Inventory

| Service | Project | Type | Cost/month |
|---|---|---|---|
| EC2 | devops-project | t2.micro | £0 (free tier) |
| Route 53 | devops-project | Hosted Zone | ~£0.50 |

## Cost Summary
**Current monthly estimate: ~£0.50**

## Useful Commands
```bash
# List all EC2 instances
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key==`Name`].Value]' \
  --output table

# Check free tier usage
aws ce get-cost-and-usage \
  --time-period Start=2025-01-01,End=2025-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost
```
