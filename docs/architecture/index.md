# Architecture Diagrams

## DevOps Project — AWS Stack

Full architecture showing AWS infrastructure and Docker Compose internals.

```mermaid
graph TB
    User -->|HTTPS/HTTP| DNS[Route 53\ncarlharvey.dev]
    DNS --> EIP[Elastic IP]
    EIP -->|Port 5000| Flask
    EIP -->|Port 3000| Grafana
    EIP -->|Port 9090| Prometheus

    subgraph AWS Free Tier - eu-west-1
        subgraph EC2 t2.micro - Ubuntu
            subgraph Docker Compose Stack
                Flask[Flask App\nweb:5000\nPython API]
                DB[(PostgreSQL\ndb:5432\nInternal only)]
                Prometheus[Prometheus\nprometheus:9090\nMetrics collector]
                Grafana[Grafana\ngrafana:3000\nDashboards]

                Flask <-->|container network| DB
                Prometheus -->|scrapes :5000/metrics| Flask
                Grafana -->|PromQL queries| Prometheus
            end
        end
        SG[Security Group\n5000 open\n3000 open\n9090 open\n22 SSH only]
        EIP
    end
```

---

## Docker Network — How Containers Communicate

Containers talk to each other using service names as hostnames. This is why Flask connects to `db:5432` not `localhost:5432`.

```mermaid
graph LR
    subgraph Docker Bridge Network
        Flask -->|db:5432| DB
        Prometheus -->|web:5000/metrics| Flask
        Grafana -->|prometheus:9090| Prometheus
    end

    subgraph External
        Internet -->|5000| Flask
        Internet -->|3000| Grafana
        Internet -->|9090| Prometheus
        Internet -.->|5432 BLOCKED| DB
    end
```

---

## Home Server — VM Layout

```mermaid
graph TB
    HW[Physical Server]-->HV[Hypervisor]
    HV-->VM1[immich-vm\nPhoto Management]
    HV-->VM2[openclaw-vm\nOpenClaw]
    HV-->VM3[hexos-vm\nHexOS]

    subgraph Local Network
        HW
        HV
        VM1
        VM2
        VM3
    end
```

---

## Future State — CI/CD Pipeline

Planned addition once GitHub Actions is configured.

```mermaid
graph LR
    Dev[Developer\npushes code] --> GH[GitHub\nmain branch]
    GH -->|triggers| GA[GitHub Actions\nCI/CD workflow]
    GA -->|runs tests| Test[pytest]
    Test -->|on pass| Deploy[SSH deploy\nto EC2]
    Deploy -->|docker compose\nup --build| EC2[EC2 Instance]
```
