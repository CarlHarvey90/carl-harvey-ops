# Home Server Infrastructure

## Hardware
| Spec | Value |
|---|---|
| CPU | |
| RAM | |
| Storage | |
| Hypervisor | Proxmox / ESXi |

## Network
- Local subnet: 192.168.x.0/24
- Remote access: Tailscale

## VM Summary

| VM | vCPUs | RAM | Disk | IP | Status |
|---|---|---|---|---|---|
| immich-vm | 2 | 4GB | 100GB | 192.168.x.x | 🟢 running |
| openclaw-vm | 1 | 2GB | 20GB | 192.168.x.x | 🟢 running |
| hexos-vm | 1 | 2GB | 20GB | 192.168.x.x | 🟢 running |
