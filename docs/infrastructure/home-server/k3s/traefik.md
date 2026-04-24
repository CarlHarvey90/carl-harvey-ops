# Deploying Traefik as ingress controller on k3s (bare metal)

**Stack:** k3s · MetalLB · Helm  
**Traefik version:** latest stable via `traefik/traefik` Helm chart  
**Prerequisite:** MetalLB deployed and IP pool active

---

## Overview

k3s ships with Traefik built in, but disabling it at install time and deploying it manually via Helm gives you full control over the configuration. This runbook covers that approach — clean Helm install, LoadBalancer service backed by MetalLB, and a whoami test to confirm end-to-end routing works before attaching real services.

Traefik sits at the edge of the cluster. It holds a single external IP (assigned by MetalLB) and routes HTTP/HTTPS traffic to the correct pods based on hostname rules in `Ingress` or `IngressRoute` resources.

---

## Prerequisites

- k3s cluster with Traefik disabled at install (`--disable=traefik`)
- MetalLB deployed with an active IP pool
- Helm installed on your local machine
- kubectl configured and pointing at your cluster

### Install Helm (Arch-based Linux)

```bash
sudo pacman -S helm
```

### Install Helm (other Linux)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## Step 1 — Add the Traefik Helm repo

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

---

## Step 2 — Create namespace

```bash
kubectl create namespace traefik
```

---

## Step 3 — Create values file

Save as `infra/traefik/values.yaml` in your repo:

```yaml
deployment:
  replicas: 1

service:
  type: LoadBalancer

ports:
  web:
    port: 80
    exposedPort: 80
  websecure:
    port: 443
    exposedPort: 443

ingressRoute:
  dashboard:
    enabled: true

additionalArguments:
  - "--api.insecure=true"
  - "--log.level=INFO"

tolerations: []

nodeSelector:
  kubernetes.io/os: linux
```

---

## Step 4 — Install Traefik

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --values infra/traefik/values.yaml
```

Wait for the rollout to complete:

```bash
kubectl rollout status deployment/traefik -n traefik
```

Confirm Traefik has received an external IP from MetalLB:

```bash
kubectl get svc traefik -n traefik
```

The `EXTERNAL-IP` column should show an IP from your MetalLB pool. Note this IP — it is the single entry point for all ingress traffic.

---

## Step 5 — Verify with a test ingress

Deploy the `whoami` container and an `Ingress` resource to confirm Traefik is routing correctly:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: default
spec:
  selector:
    app: whoami
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: whoami.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f infra/traefik/test-ingress.yaml
```

Test routing using a Host header override (no DNS required):

```bash
curl -H "Host: whoami.local" http://<traefik-external-ip>
```

Expected response: a block of text from the whoami container showing the request hostname, IP, and headers. This confirms Traefik received the request and routed it to the correct pod based on the hostname rule.

Clean up after verification:

```bash
kubectl delete -f infra/traefik/test-ingress.yaml
```

---

## How routing works

Each service that needs external access gets an `Ingress` resource with a `host` rule. Traefik watches the Kubernetes API for these resources and updates its routing table automatically — no restarts or manual config changes needed.

During development without external DNS, test with the `-H "Host: ..."` curl flag to spoof the hostname. When DNS is configured (a wildcard record pointing at the Traefik IP), real hostnames resolve automatically.

---

## Troubleshooting

**Traefik service stuck on `<pending>` for EXTERNAL-IP**

MetalLB is not assigning an IP. Check MetalLB pods are running and the IP pool resource exists:

```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
```

**curl returns connection refused**

Traefik pod is not running or the IP is wrong:

```bash
kubectl get pods -n traefik
kubectl get svc traefik -n traefik
```

**curl returns 404**

Traefik is running but no route matched. The `host` value in your Ingress must match the `-H` header exactly:

```bash
kubectl get ingress -A
```

**whoami pod not responding**

Check the pod scheduled correctly and inspect events if not:

```bash
kubectl get pods -n default
kubectl describe pod whoami -n default
```

---

## Next steps

With Traefik in place, the next components in the sequence are:

- **Longhorn** — CSI storage driver for persistent volumes
- **Harbor** — private container registry
- **ArgoCD** — GitOps deployment controller
