# 🏛️ Project Olympus - Production Homelab Infrastructure

Greek mythology-themed self-hosted infrastructure with automated health monitoring and zero-trust remote access.

![Project Status](https://img.shields.io/badge/status-active-success)
![Monitoring](https://img.shields.io/badge/monitoring-automated-blue)
![Platform](https://img.shields.io/badge/platform-Proxmox%20VE-orange)

## 🎯 Overview

Project Olympus is a production-grade homelab infrastructure running on Proxmox VE, featuring:
- **Automated health monitoring** with real-time Discord alerts
- **Zero-trust remote access** via Twingate (no port forwarding)
- **Custom DNS namespace** (olympus.lab) via Pi-hole
- **Containerized services** using LXC and Docker
- **Cloud storage** using Nextcloud

## 🏗️ Architecture
```
                ┌─────────────────────────────────────┐
                │      PROXMOX VE (10.0.0.200)        │
                │    ThinkPad X1 Carbon Gen 5         │
                └──────────────┬──────────────────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
    ┌───────────▼────┐  ┌──────▼──────┐  ┌──▼──────────┐
    │   Internet     │  │   Clients   │  │   Remote    │
    │   (WAN)        │  │  (LAN)      │  │  (Twingate) │
    └───────┬────────┘  └──────┬──────┘  └──────┬──────┘
            │                  │                 │
            │                  └────────┬────────┘
            │                           │
            │                  ┌────────▼────────┐
            │                  │  DNS: 10.0.0.100│
            │                  └────────┬────────┘
            │                           │
            │              All *.olympus.lab queries
            │              resolve to 10.0.0.104
            │                           │
    ┌───────▼───────────────────────────▼──────────────────────┐
    │                                                          │
    │  ┌──────────────┐          ┌───────────────┐             │
    │  │   ATHENA     │          │   CERBERUS    │             │
    │  │  (LXC 100)   │◄─────────┤   (LXC 104)   │             │
    │  │  10.0.0.100  │  Proxies │   10.0.0.104  │             │
    │  │              │   :80    │               │             │
    │  │  Pi-hole DNS │          │  NPM Proxy    │             │
    │  │  Ad Blocking │          │  SSL/TLS      │             │
    │  └──────────────┘          └───────┬───────┘             │
    │         ▲                           │                    │
    │         │                           │                    │
    │         │                  ┌────────┼─────────────┐      │
    │         │                  │        │        │    │      │
    │  ┌──────┴──────┐   ┌───────▼───┐ ┌─▼──────┐ ┌───▼────┐   │
    │  │ HEPHAESTUS  │   │  HERMES   │ │ ARGUS  │ │ARTEMIS │   │
    │  │  (LXC 101)  │   │ (LXC 102) │ │(LXC103)│ │(LXC105)│   │
    │  │  10.0.0.101 │   │10.0.0.102 │ │10.0.0. │ │10.0.0. │   │
    │  │             │   │           │ │  103   │ │  105   │   │
    │  │  Portainer  │   │    N8N    │ │ Uptime │ │Nextcloud│  │
    │  │  :9000      │   │ Workflows │ │  Kuma  │ │+ PostSQL│  │
    │  └─────────────┘   │   :5678   │ │ :3001  │ │ :8080  │   │
    │         ▲          └─────┬─────┘ └────▲───┘ └────────┘   │
    │         │                │            │                  │
    │         └────────────────┴────────────┘                  │
    │          Manages all     Health checks every 5 min       │
    │          containers      (6 services monitored)          │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
LEGEND:
─────►  Network flow / Connection
◄─────  Reverse proxy routing
Clients query DNS (Athena) → DNS returns Cerberus IP →
Cerberus proxies to backend services with SSL termination
Service Stack:
├─ DNS & Ad-blocking: Pi-hole (Athena)
├─ Container Management: Portainer (Hephaestus)
├─ Workflow Automation: N8N (Hermes)
├─ Uptime Monitoring: Uptime Kuma (Argus)
├─ Reverse Proxy & SSL: Nginx Proxy Manager (Cerberus)
└─ File Storage: Nextcloud + PostgreSQL (Artemis)
Monitoring Stack:
├─ N8N: Real-time alerts (Every 5 min → Discord)
└─ Uptime Kuma: Visual dashboard (6 services, 99%+ uptime)
```

## 🛡️ Services

## Infrastructure Overview

| Service | Container | IP | Ports | Purpose | URL |
|---------|-----------|-----|-------|---------|-----|
| **Athena** | LXC 100 | 10.0.0.100 | 80, 53 | Pi-hole DNS & Ad Blocking | https://athena.olympus.lab/admin/ |
| **Hephaestus** | LXC 101 | 10.0.0.101 | 9000 | Portainer Container Management | http://10.0.0.101:9000/ |
| **Hermes** | LXC 102 | 10.0.0.102 | 5678 | N8N Workflow Automation | https://hermes.olympus.lab/ |
| **Argus** | LXC 103 | 10.0.0.103 | 3001 | Uptime Kuma Monitoring | https://argus.olympus.lab/ |
| **Cerberus** | LXC 104 | 10.0.0.104 | 80, 443, 81 | Nginx Proxy Manager | http://cerberus.olympus.lab:81 |
| **Artemis** | LXC 105 | 10.0.0.105 | 8080 | Nextcloud File Storage | https://artemis.olympus.lab |
| **Olympus** | Proxmox Host | 10.0.0.200 | 8006 | Proxmox VE Hypervisor | https://10.0.0.200:8006 |


## 📸 Screenshots

### N8N Health Monitoring Workflow
<img width="1531" height="811" alt="Hermes (6 services monitoring)" src="https://github.com/user-attachments/assets/6d2e4588-9a97-4fae-8d2c-ac17cd8b932d" />
Parallel health checks with intelligent failure detection and Discord alerting

### Discord Alert Example
<img width="1411" height="582" alt="Alert message in discord" src="https://github.com/user-attachments/assets/d132001d-7fcc-4b18-b237-d82fb44709f0" />
Real-time notifications with service name and error details

### Pi-hole Dashboard
<img width="1672" height="1007" alt="Athena(pihole)" src="https://github.com/user-attachments/assets/bff01864-d54b-474e-abe6-e842acd699ed" />
Network-wide DNS filtering with custom local domain resolution

### Portainer Container Management
<img width="1672" height="996" alt="Hephaestus(portainer)" src="https://github.com/user-attachments/assets/5ed931c7-9386-4b97-a1ef-c3b705ba9bce" />
Docker container orchestration and monitoring

### Uptime Kuma Dashboard
<img width="1671" height="872" alt="Argus (Uptime Kuma)" src="https://github.com/user-attachments/assets/ea876a72-535c-4f1b-b24b-a4e89f4cba64" />
Visual monitoring dashboard with uptime percentages and response time graphs

### Nginxy Proxy Manager Dashboard
<img width="1667" height="960" alt="Cerberus (Nginx proxy manager)" src="https://github.com/user-attachments/assets/5a7b079d-af9e-4712-9371-3e04e6c6607d" />
Reverse proxy maanger

### NextCloud Dashboard
<img width="1675" height="928" alt="Artemis (Nextcloud)" src="https://github.com/user-attachments/assets/44486203-43f2-4263-b023-e58bf23b81da" />
Cloud Storage dashboard


## ✨ Features

### Automated Health Monitoring
- **Parallel health checks** every 5 minutes
- **Real-time Discord alerts** on service failures
- **Detailed error reporting** with service names and error codes
- **Smart failure detection** supporting multiple simultaneous failures

### Zero-Trust Security
- **Twingate connector** for secure remote access
- **No port forwarding** required
- **No traditional VPN** overhead
- **Per-service access control**

### Custom DNS
- **Local DNS namespace** (olympus.lab)
- **Network-wide ad blocking** via Pi-hole
- **Local service discovery** with human-readable names

### HTTPS & SSL/TLS
- **Encrypted connections**: All services use HTTPS
- **Wildcard certificate**: Single cert covers all *.olympus.lab domains
- **Self-signed PKI**: Internal certificate authority for homelab
- **Force SSL**: Automatic HTTP → HTTPS redirect
- **Valid for 1 year**: Certificate expires March 29, 2027

### Reverse Proxy & Clean URLs (Cerberus)
- **No port numbers**: Services accessible via clean URLs
- **Centralized routing**: All HTTP/HTTPS traffic through Cerberus (10.0.0.104)
- **SSL termination**: HTTPS handled at proxy layer
- **DNS integration**: Pi-hole routes all *.olympus.lab to proxy

### Automated Monitoring (Hermes + Argus)
- **6 services monitored**: All infrastructure tracked
- **HTTPS health checks**: Monitors encrypted endpoints
- **SSL-aware**: Ignores self-signed certificate warnings
- **Visual dashboard**: Real-time status in Uptime Kuma
- **99%+ uptime**: Continuous monitoring ensures availability

### Network Architecture
- **Internal DNS**: Pi-hole provides local domain resolution (*.olympus.lab)
- **Remote access**: Twingate zero-trust network access
- **Containerized**: All services run in isolated LXC containers
- **Proxmox backend**: Enterprise-grade virtualization platform


## 💾 Backup & Disaster Recovery

### Automated Backup System
- **Frequency:** Daily at 2:00 AM
- **Method:** Proxmox vzdump (snapshot mode)
- **Compression:** ZSTD
- **Retention:** 7 days (rolling)
- **Storage:** Local Proxmox storage (`/var/lib/vz/dump/`)
- **Total Size:** 4.6 GB per backup set (~32 GB maximum)

### Backup Coverage
| Container | Backup Size | Critical Level | RPO |
|-----------|-------------|----------------|-----|
| Athena (DNS) | 290 MB | ⭐⭐⭐⭐⭐ | 24h |
| Hephaestus (Portainer) | 506 MB | ⭐⭐ | 24h |
| Hermes (N8N) | 881 MB | ⭐⭐⭐⭐ | 24h |
| Argus (Uptime Kuma) | 637 MB | ⭐⭐⭐⭐ | 24h |
| Cerberus (Nginx Proxy) | 1.1 GB | ⭐⭐⭐⭐⭐ | 24h |
| Artemis (Nextcloud) | 1.3 GB | ⭐⭐⭐⭐⭐ | 24h |

### Recovery Metrics
- **RTO (Recovery Time Objective):** 2 hours target, **2 minutes actual**
- **RPO (Recovery Point Objective):** 24 hours
- **Last Tested:** April 11, 2026 (Hephaestus full restore - successful)
- **Data Integrity:** 100% (verified via test restore)

### Monitoring & Alerting
- **N8N workflow** runs daily at 8:00 AM
- **Python-based validation** checks for 6 backup files
- **Discord alerts** on backup failure
- **Detection window:** Maximum 6 hours (between backup run and check)

### Disaster Recovery Procedures
**Full Container Restore:**
```bash
# Stop container (if running)
pct stop <CTID>

# Restore from backup
pct restore <CTID> /var/lib/vz/dump/vzdump-lxc-<CTID>-<timestamp>.tar.zst

# Start restored container
pct start <CTID>

# Verify functionality
pct status <CTID>
```

**Verified Recovery Time:** 2 minutes (tested with LXC 101)

## ⚠️ Security Notice

**This repository contains sanitized configurations.**

All sensitive data has been replaced with placeholders:
- IP addresses → Use your own network range
- Discord webhooks → Create your own webhook
- Credentials → Never commit real credentials

## 📊 System Specs

**Hardware:**
- ThinkPad X1 Carbon Gen 5
- Intel i5-7300U
- 16GB RAM
- 500GB SSD

**Software:**
- Proxmox VE 8.x
- Debian 12 (LXC containers)
- Docker 29.3.1
- N8N (latest)


## 📖 Lessons Learned

See [docs/lessons-learned.md](docs/lessons-learned.md) for challenges encountered and solutions.

**Note:** This is a living project. Documentation and configurations are continuously updated.
