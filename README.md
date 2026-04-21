# 🏛️ Project Olympus - Production Homelab Infrastructure

Greek mythology-themed self-hosted infrastructure with automated health monitoring and zero-trust remote access.

![Project Status](https://img.shields.io/badge/status-active-success)
![Monitoring](https://img.shields.io/badge/monitoring-automated-blue)
![Platform](https://img.shields.io/badge/platform-Proxmox%20VE-orange)
![Services](https://img.shields.io/badge/services-7%2F10-yellow)

## 🎯 Overview

Project Olympus is a production-grade homelab infrastructure running on Proxmox VE, featuring:
- **Full observability stack** with Prometheus, Grafana, and real-time Discord alerts
- **Zero-trust remote access** via Twingate (no port forwarding)
- **Custom DNS namespace** (olympus.lab) via Pi-hole
- **Containerized services** using LXC and Docker
- **Automated backups** with tested disaster recovery (2-minute RTO)

## 🛡️ Services

| Service | Container | IP | Purpose | URL |
|---------|-----------|-----|---------|-----|
| **Athena** | LXC 100 | 10.0.0.100 | Pi-hole DNS & Ad Blocking | https://athena.olympus.lab/admin |
| **Hephaestus** | LXC 101 | 10.0.0.101 | Portainer Container Management | http://10.0.0.101:9000 |
| **Hermes** | LXC 102 | 10.0.0.102 | N8N Workflow Automation | https://hermes.olympus.lab |
| **Argus** | LXC 103 | 10.0.0.103 | Uptime Kuma Monitoring | https://argus.olympus.lab |
| **Cerberus** | LXC 104 | 10.0.0.104 | Nginx Proxy Manager + SSL | http://cerberus.olympus.lab:81 |
| **Artemis** | LXC 105 | 10.0.0.105 | Nextcloud + PostgreSQL | https://artemis.olympus.lab |
| **Apollo** | LXC 106 | 10.0.0.106 | Prometheus + Grafana | https://apollo.olympus.lab |
| **Olympus** | Host | 10.0.0.200 | Proxmox VE Hypervisor | https://10.0.0.200:8006 |

## 🏗️ Architecture

```
                ┌─────────────────────────────────────┐
                │      PROXMOX VE (10.0.0.200)        │
                │    ThinkPad X1 Carbon Gen 5         │
                └──────────────┬──────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
          Internet          Clients          Twingate
           (WAN)             (LAN)           (Remote)
              │                └───────┬────────┘
              │                        │
              │               ┌────────▼────────┐
              │               │  Athena (DNS)   │
              │               │   10.0.0.100    │
              │               └────────┬────────┘
              │                        │ *.olympus.lab → 10.0.0.104
              └────────────────────────▼
                        ┌──────────────────┐
                        │    Cerberus      │
                        │   10.0.0.104     │
                        │  Reverse Proxy   │
                        │  SSL Termination │
                        └──────┬───────────┘
                               │
          ┌──────────┬─────────┼─────────┬──────────┬──────────┐
          │          │         │         │          │          │
       Athena  Hephaestus   Hermes     Argus     Artemis    Apollo
      (DNS)   (Portainer)  (N8N)  (Uptime Kuma)(Nextcloud)(Prometheus
                                                          + Grafana)

Monitoring Flow:
Node Exporter (all 7 containers, :9100)
        ↓ scraped every 15s
Prometheus (:9090)
        ↓ alert rules fire
Alertmanager (:9093)
        ↓
Discord #olympus-alerts
        +
Grafana (:3000) → https://apollo.olympus.lab
```

## 📊 Monitoring Stack (Apollo)

**Prometheus + Grafana + Alertmanager deployed across all 7 containers.**

Node Exporter runs on every container. Prometheus scrapes metrics every 15 seconds.
Grafana visualizes via Node Exporter Full dashboard (ID: 1860).

**Alert rules:**
| Alert | Condition | Severity |
|-------|-----------|----------|
| ContainerDown | Target unreachable > 1min | Critical |
| HighCPUUsage | CPU > 80% for 2min | Warning |
| HighMemoryUsage | Memory > 85% for 2min | Warning |
| LowDiskSpace | Disk < 15% for 5min | Warning |
| CriticalDiskSpace | Disk < 5% for 1min | Critical |

## 💾 Backup & Disaster Recovery

- **Schedule:** Daily 2:00 AM via Proxmox vzdump
- **Compression:** ZSTD | **Retention:** 7 days | **Size:** ~4.6GB per set
- **RTO Target:** 2 hours | **Actual RTO:** 2 minutes (tested April 11, 2026)
- **Monitoring:** N8N validates backup count daily at 8:00 AM → Discord alert on failure

**Restore command:**
```bash
pct stop <CTID>
pct restore <CTID> /var/lib/vz/dump/vzdump-lxc-<CTID>-<timestamp>.tar.zst
pct start <CTID>
```

## ✨ Key Features

- **HTTPS everywhere** - Wildcard self-signed cert covering all *.olympus.lab domains
- **Zero-trust access** - Twingate, no port forwarding, no VPN overhead
- **Full observability** - Metrics, dashboards, and alerting across all containers
- **Automated backups** - Daily backups with Discord alerting on failure
- **Infrastructure as code** - All configs documented and version controlled

## 📸 Screenshots

### N8N Health Monitoring Workflow
<img width="1531" height="811" alt="Hermes (6 services monitoring)" src="https://github.com/user-attachments/assets/6d2e4588-9a97-4fae-8d2c-ac17cd8b932d" />

### Discord Alert Example
<img width="1411" height="582" alt="Alert message in discord" src="https://github.com/user-attachments/assets/d132001d-7fcc-4b18-b237-d82fb44709f0" />

### Pi-hole Dashboard
<img width="1672" height="1007" alt="Athena(pihole)" src="https://github.com/user-attachments/assets/bff01864-d54b-474e-abe6-e842acd699ed" />

### Portainer Container Management
<img width="1672" height="996" alt="Hephaestus(portainer)" src="https://github.com/user-attachments/assets/5ed931c7-9386-4b97-a1ef-c3b705ba9bce" />

### Uptime Kuma Dashboard
<img width="1671" height="872" alt="Argus (Uptime Kuma)" src="https://github.com/user-attachments/assets/ea876a72-535c-4f1b-b24b-a4e89f4cba64" />

### Nginx Proxy Manager
<img width="1667" height="960" alt="Cerberus (Nginx proxy manager)" src="https://github.com/user-attachments/assets/5a7b079d-af9e-4712-9371-3e04e6c6607d" />

### Nextcloud Dashboard
<img width="1675" height="928" alt="Artemis (Nextcloud)" src="https://github.com/user-attachments/assets/44486203-43f2-4263-b023-e58bf23b81da" />

## 📊 System Specs

| Component | Detail |
|-----------|--------|
| Hardware | ThinkPad X1 Carbon Gen 5 |
| CPU | Intel i5-7300U |
| RAM | 16GB |
| Storage | 500GB SSD |
| Hypervisor | Proxmox VE 8.x |
| Container OS | Debian 12 / Ubuntu 24.04 |

## ⚠️ Security Notice

All sensitive data has been replaced with placeholders. Never commit real credentials, webhook URLs, or IP ranges.

## 📖 Lessons Learned

See [docs/lessons-learned.md](docs/lessons-learned.md) for challenges and solutions encountered during deployment.