# SOC-lite DevSecOps Project

## Overview
Self-hosted Security Operations Center on k3s (WSL) using plain YAML manifests.
All services run in a single k3s cluster, orchestrated by n8n.

## Stack
- **k3s** — single-node Kubernetes (WSL)
- **n8n** — workflow automation / alerting engine
- **Prometheus** — metrics collection
- **Grafana** — dashboards
- **Loki** — log aggregation
- **CrowdSec** — threat detection (brute force, scanning)
- **Traefik** — reverse proxy (k3s default)
- **Discord** — alert channel (via n8n webhook)

## Directory Structure
```
n8n-k8s/
├── AGENTS.md              # this file
├── base/                  # shared resources (namespace, secrets)
├── n8n/                   # n8n deployment, service, ingress
├── monitoring/
│   ├── prometheus/
│   ├── grafana/
│   └── loki/
├── security/
│   └── crowdsec/
└── alerts/                # n8n workflow JSON exports
```

## Phases

### Phase 1: Foundation
- [x] n8n deployed and running
- [x] Fix ingress (HTTP→HTTPS redirect removed, TLS cert added)
- [x] Verify end-to-end access from Windows browser

### Phase 2: Monitoring
- [x] Deploy Loki for log aggregation
- [x] Deploy Grafana (dashboards for metrics + logs)
- [x] Connect Loki to Grafana
- [ ] n8n workflow: Prometheus webhook → alert on traffic spike

### Phase 3: Security (current)
- [x] Deploy CrowdSec (parse Traefik logs, detect threats)
- [x] Traefik configured to write JSON logs to /var/log/traefik/traefik.log
- [x] n8n workflow: CrowdSec alert → Discord notification
- [ ] Add GeoIP enrichment to alerts

### Phase 4: Automation
- [ ] n8n workflow: log anomaly → auto-block IP via Traefik
- [ ] n8n workflow: daily security summary email
- [ ] Prometheus → n8n → auto-scale alerts

## Current Status
- **n8n**: Running at https://n8n.kube (self-signed cert)
- **Traefik**: Running, HTTP→HTTPS redirect disabled for local dev
- **WSL IP**: 172.30.60.115
- **Next**: Add GeoIP enrichment to alerts, or move to Phase 4

## Conventions
- No Helm charts — plain YAML manifests only
- One namespace per concern (`n8n`, `monitoring`, `security`)
- Use Kustomize overlays only if absolutely necessary

## Learning Approach
- **Hands-on learning**: User writes the code, I guide
- **Only point out mistakes** — don't fix files for them
- **Guide step by step** — one component at a time
- **Update this file** to track progress
