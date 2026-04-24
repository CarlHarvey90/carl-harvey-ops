# Deploying MetalLB on k3s (bare metal)

**Stack:** k3s · MetalLB v0.14.5  
**Mode:** L2 (ARP)  
**Prerequisite:** k3s cluster running, kubectl configured locally

---

## Overview

Kubernetes on bare metal has no cloud provider to assign external IPs to `LoadBalancer` services. MetalLB fills that gap by watching for `LoadBalancer` service requests and assigning IPs from a pool you define, advertising them on your LAN via L2 ARP.

Every component that needs external reachability — ingress controllers, registries, application services — depends on MetalLB being in place first. It is the first thing to deploy on a bare metal k3s cluster.

---

## Prerequisites

- k3s cluster with at least one worker node
- kubectl installed and configured on your local machine pointing at the cluster
- A free IP range on your LAN subnet that is outside your router's DHCP assignment range

### kubectl setup (if not already configured)

Copy the kubeconfig from your k3s control plane node. Do not create `.kube/config` as a directory — it must be a file:

```bash
mkdir -p ~/.kube
scp user@<control-plane-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

Replace the loopback address k3s writes by default with your control plane's actual IP:

```bash
sed -i 's/127.0.0.1/<control-plane-ip>/g' ~/.kube/config
```

Verify:

```bash
kubectl get nodes
```

All nodes should show `Ready`.

---

## Step 1 — Pre-flight checks

Confirm the cluster is healthy before applying anything:

```bash
kubectl get nodes
kubectl get pods -A
```

No pods should be in `CrashLoopBackOff` or `Error` state.

---

## Step 2 — Deploy MetalLB

Apply the official manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

Wait for MetalLB pods to become ready:

```bash
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

Verify:

```bash
kubectl get pods -n metallb-system
```

Expected: one `controller` pod and one `speaker` pod per node, all `Running`.

---

## Step 3 — Configure IP address pool

Choose a free IP range on your LAN subnet. Check your router's DHCP settings first — the range you choose must not overlap with any addresses your router assigns automatically. A conflict will cause intermittent and difficult-to-debug failures.

Save as `infra/metallb/ip-address-pool.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: main-pool
  namespace: metallb-system
spec:
  addresses:
    - <start-ip>-<end-ip>
```

Save as `infra/metallb/l2-advertisement.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: main-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - main-pool
```

Apply both:

```bash
kubectl apply -f infra/metallb/ip-address-pool.yaml
kubectl apply -f infra/metallb/l2-advertisement.yaml
```

---

## Step 4 — Verify with a test service

Deploy a minimal nginx pod and a `LoadBalancer` service to confirm MetalLB assigns an IP:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  namespace: default
  labels:
    app: nginx-test
spec:
  containers:
    - name: nginx
      image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: nginx-test
  ports:
    - port: 80
      targetPort: 80
```

Apply and watch for IP assignment:

```bash
kubectl apply -f infra/metallb/test-service.yaml
kubectl get svc nginx-test -w
```

The `EXTERNAL-IP` column should change from `<pending>` to an address from your pool within 30 seconds.

Confirm the IP is reachable:

```bash
curl http://<assigned-ip>
```

Expected: nginx welcome page HTML.

Clean up after verification:

```bash
kubectl delete -f infra/metallb/test-service.yaml
```

---

## Repo structure

```
infra/
  metallb/
    ip-address-pool.yaml
    l2-advertisement.yaml
    test-service.yaml
```

Commit the pool and advertisement manifests. The test service can be kept for reference or removed.

---

## Troubleshooting

**External IP stays as `<pending>`**

MetalLB speakers may not be running on all nodes, or the pool resource was not applied correctly:

```bash
kubectl get pods -n metallb-system -o wide
kubectl get ipaddresspool -n metallb-system
```

Also verify the IP range does not conflict with your router's DHCP pool.

**`kubectl get nodes` returns connection refused**

The server address in `~/.kube/config` is likely still set to `127.0.0.1`. Confirm it matches your control plane's actual IP:

```bash
grep server ~/.kube/config
```

**Permission denied copying kubeconfig**

The k3s.yaml file on the control plane is root-owned. Either copy it with sudo on the remote side first, or use a user with sufficient permissions.

---

## Next steps

With MetalLB in place, the next components in the sequence are:

- **Traefik** — ingress controller, uses MetalLB to get its external IP
- **Longhorn** — CSI storage driver for persistent volumes
- **Harbor** — private container registry
