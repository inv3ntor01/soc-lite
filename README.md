# 🛡️ SOC-lite - Security Operations Center on k3s

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.31-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![k3s](https://img.shields.io/badge/k3s-Lightweight-FF6B35?style=flat-square&logo=k3s&logoColor=white)](https://k3s.io)
[![CrowdSec](https://img.shields.io/badge/CrowdSec-Threat%20Detection-EA3C1E?style=flat-square&logo=crowdsec&logoColor=white)](https://www.crowdsec.net)
[![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800?style=flat-square&logo=grafana&logoColor=white)](https://grafana.com)
[![Loki](https://img.shields.io/badge/Loki-Logs-F5BD42?style=flat-square&logo=grafana&logoColor=white)](https://grafana.com/oss/loki)
[![n8n](https://img.shields.io/badge/n8n-Automation-EA4B71?style=flat-square&logo=n8n&logoColor=white)](https://n8n.io)

A lightweight, self-hosted Security Operations Center (SOC) built on k3s using plain YAML manifests. No Helm charts, no complexity — just Kubernetes fundamentals.

![Architecture](https://img.shields.io/badge/Architecture-Blue%20Team-6366F1?style=for-the-badge)

## 📋 Overview

SOC-lite provides real-time threat detection, log aggregation, and automated alerting for your Kubernetes workloads. Perfect for learning security operations, home labs, or small production environments.

### What's Included

| Component | Purpose | Status |
|-----------|---------|--------|
| 🔒 **CrowdSec** | Threat detection (brute force, scanning, CVEs) | ✅ Deployed |
| 📊 **Grafana** | Dashboards for metrics + logs | ✅ Deployed |
| 📝 **Loki** | Centralized log aggregation | ✅ Deployed |
| 🤖 **n8n** | Workflow automation + Discord alerts | ✅ Deployed |
| 🌐 **Traefik** | Reverse proxy + ingress controller | ✅ Deployed |
| 📈 **Prometheus** | Metrics collection | 🔜 Coming Soon |

## 🏗️ Architecture

```
                    Internet
                       │
                  ┌────▼────┐
                  │ Traefik │
                  │ (Ingress)│
                  └────┬────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   ┌────▼────┐   ┌─────▼─────┐   ┌────▼────┐
   │   n8n   │   │  CrowdSec  │   │ Grafana │
   │ (Alerts)│   │ (Security) │   │(Dashboard)│
   └────┬────┘   └─────┬─────┘   └────┬────┘
        │              │              │
        │         ┌────▼────┐         │
        └────────►│  Loki   │◄────────┘
                  │  (Logs) │
                  └─────────┘
```

### Data Flow

1. **Traefik** routes traffic → writes JSON logs to `/var/log/traefik/`
2. **CrowdSec** reads logs → detects threats (brute force, CVEs, scanning)
3. **CrowdSec** triggers alerts → sends to **n8n** via webhook
4. **n8n** processes alerts → sends notifications to **Discord**
5. **Loki** aggregates all logs → **Grafana** visualizes dashboards

## 🚀 Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| **WSL** | 2 | Windows Subsystem for Linux |
| **k3s** | Latest | Single-node Kubernetes |
| **kubectl** | Latest | Kubernetes CLI |
| **OpenSSL** | Latest | For TLS certificate generation |

### System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Disk | 20 GB | 50 GB |

## 📦 Installation

### Step 1: Install k3s

```bash
curl -sfL https://get.k3s.io | sh -
```

### Step 2: Verify k3s

```bash
kubectl get nodes
kubectl get pods -A
```

### Step 3: Create Namespaces

```bash
kubectl create namespace n8n
kubectl create namespace monitoring
kubectl create namespace security
```

### Step 4: Deploy n8n

```bash
kubectl apply -f base/namespace.yaml
kubectl apply -f n8n/
```

### Step 5: Generate TLS Certificates

```bash
# Create certs directory
mkdir -p certs

# Generate self-signed certificates
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/n8n.kube.key \
  -out certs/n8n.kube.crt \
  -subj "/CN=n8n.kube"

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/loki.kube.key \
  -out certs/loki.kube.crt \
  -subj "/CN=loki.kube"

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/grafana.kube.key \
  -out certs/grafana.kube.crt \
  -subj "/CN=grafana.kube"
```

### Step 6: Deploy Loki

```bash
kubectl apply -f monitoring/loki/
```

### Step 7: Deploy Grafana

```bash
kubectl apply -f monitoring/grafana/
```

### Step 8: Deploy CrowdSec

```bash
kubectl apply -f security/crowdsec/
```

### Step 9: Configure Traefik for JSON Logs

```bash
kubectl patch deployment traefik -n traefik --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--accesslog.filepath=/var/log/traefik/traefik.log"},
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--accesslog.format=json"},
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts","value":[{"name":"traefik-logs","mountPath":"/var/log/traefik"}]},
  {"op":"add","path":"/spec/template/spec/volumes","value":[{"name":"traefik-logs","hostPath":{"path":"/var/log/traefik","type":"DirectoryOrCreate"}}]}
]'
```

### Step 10: Configure Discord Alerts

1. **Create Discord Webhook:**
   - Open Discord → Server Settings → Integrations → Webhooks
   - Click "New Webhook" → Copy URL

2. **Update CrowdSec Notification Config:**
   - Edit `security/crowdsec/crowdsec-notifications.yaml`
   - Replace `http://n8n.n8n.svc.cluster.local:5678/webhook/crowdsec-alert` with your webhook path

3. **Create n8n Workflow:**
   - Open n8n at `https://n8n.kube`
   - Add Webhook node → Set path to `crowdsec-alert`
   - Add Discord node → Paste your webhook URL
   - Connect nodes → Activate workflow

### Step 11: Configure DNS (Windows)

Add to `C:\Windows\System32\drivers\etc\hosts`:

```
172.30.60.115 n8n.kube
172.30.60.115 loki.kube
172.30.60.115 grafana.kube
```

## 🌐 Access Points

| Service | URL | Default Credentials |
|---------|-----|---------------------|
| **n8n** | https://n8n.kube | Setup on first login |
| **Grafana** | https://grafana.kube | admin / admin |
| **Loki** | https://loki.kube | - |
| **CrowdSec API** | http://localhost:8080 | - |

## 📁 Directory Structure

```
n8n-k8s/
├── AGENTS.md                    # Project tracking
├── README.md                    # This file
├── base/                        # Shared resources
│   └── namespace.yaml
├── n8n/                         # n8n workflow automation
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── ingress-class.yaml
│   └── pvc.yaml
├── monitoring/
│   ├── loki/                    # Log aggregation
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   ├── pvc.yaml
│   │   └── configmap.yaml
│   └── grafana/                 # Dashboards
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       └── pvc.yaml
├── security/
│   └── crowdsec/                # Threat detection
│       ├── crowdsec-deployment.yaml
│       ├── crowdsec-pvc.yaml
│       ├── crowdsec-config.yaml
│       └── crowdsec-notifications.yaml
└── alerts/                      # n8n workflow exports
```

## 🔍 Verification

### Check All Pods

```bash
kubectl get pods -A | grep -E "(n8n|monitoring|security|traefik)"
```

### Verify CrowdSec is Reading Logs

```bash
kubectl exec -n security deploy/crowdsec -- cscli metrics show acquisition
```

### Test Discord Alerts

```bash
curl -X POST http://n8n.n8n.svc.cluster.local:5678/webhook/crowdsec-alert \
  -H "Content-Type: application/json" \
  -d '{"decisions":[{"value":"1.2.3.4","scenario":"test","duration":3600}]}'
```

## 🛠️ Troubleshooting

### CrowdSec Not Reading Logs

```bash
# Restart CrowdSec to pick up new log files
kubectl rollout restart deployment crowdsec -n security

# Check logs
kubectl logs -n security -l app=crowdsec --tail=50
```

### Traefik Not Writing Logs

```bash
# Verify Traefik has log volume mounted
kubectl get deployment traefik -n traefik -o jsonpath='{.spec.template.spec.containers[0].volumeMounts}'

# Check if log file exists
ls -la /var/log/traefik/
```

### n8n Webhook Not Working

```bash
# Test webhook from inside cluster
kubectl exec -n security deploy/crowdsec -- wget -qO- \
  --post-data='{"test":"data"}' \
  --header='Content-Type: application/json' \
  http://n8n.n8n.svc.cluster.local:5678/webhook/crowdsec-alert
```

## 📚 Learning Resources

### What You'll Learn

- **Kubernetes**: Deployments, Services, Ingress, PVCs, ConfigMaps
- **Security**: Threat detection, log analysis, IP blocking
- **Monitoring**: Log aggregation, dashboards, alerting
- **Automation**: Workflow automation with n8n
- **Networking**: Reverse proxy, TLS, DNS configuration

### Blue Team Concepts

| Concept | What You Built |
|---------|----------------|
| **SIEM** | Loki + Grafana (log aggregation + visualization) |
| **SOAR** | n8n (automated alerting) |
| **NDR** | CrowdSec (network threat detection) |
| **Threat Intel** | CrowdSec community blocklist |

### Red Team Value

Understanding blue team tools helps you:
- Know what logs you're leaving behind
- Understand detection thresholds
- Time attacks when monitoring is weakest
- Write payloads that evade detection

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## ⭐ Support

If you find this project helpful, please give it a ⭐ on GitHub!

### Buy Me a Coffee

If SOC-lite has helped you learn security operations or saved you time, consider buying me a coffee!

[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-FFDD00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://www.buymeacoffee.com/YOUR_USERNAME)

### GitHub Sponsors

You can also sponsor me through GitHub Sponsors:

[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-EA4B71?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/YOUR_USERNAME)

### Other Ways to Support

| Platform | Link |
|----------|------|
| 💰 **Ko-fi** | [![Ko-fi](https://img.shields.io/badge/Ko--fi-FF5E5B?style=flat-square&logo=ko-fi&logoColor=white)](https://ko-fi.com/YOUR_USERNAME) |
| 🎁 **Open Collective** | [![Open Collective](https://img.shields.io/badge/Open%20Collective-3385FF?style=flat-square&logo=open-collective&logoColor=white)](https://opencollective.com/YOUR_USERNAME) |
| 🐦 **Twitter** | [![Twitter](https://img.shields.io/badge/Twitter-1DA1F2?style=flat-square&logo=twitter&logoColor=white)](https://twitter.com/YOUR_USERNAME) |

> **Note:** Replace `YOUR_USERNAME` with your actual username on each platform.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [CrowdSec](https://www.crowdsec.net/) - Community-powered security
- [Grafana](https://grafana.com/) - Observability platform
- [n8n](https://n8n.io/) - Workflow automation
- [k3s](https://k3s.io/) - Lightweight Kubernetes
- [Traefik](https://traefik.io/) - Cloud-native reverse proxy

---

**Built with ❤️ for the security community**
