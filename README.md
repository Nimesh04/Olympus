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

## 🏗️ Architecture
```
┌─────────────────────────────────────┐
                    │      PROXMOX VE (10.0.0.200)       │
                    │    ThinkPad X1 Carbon Gen 5        │
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
        │                                                            │
        │  ┌──────────────┐          ┌───────────────┐             │
        │  │   ATHENA     │          │   CERBERUS    │             │
        │  │  (LXC 100)   │◄─────────┤   (LXC 104)   │             │
        │  │  10.0.0.100  │  Proxies │   10.0.0.104  │             │
        │  │              │   :80    │               │             │
        │  │  Pi-hole DNS │          │  NPM Proxy    │             │
        │  │  Ad Blocking │          │  :80 :443 :81 │             │
        │  └──────────────┘          └───────┬───────┘             │
        │         ▲                           │                     │
        │         │                           │                     │
        │         │                  ┌────────┼────────┐            │
        │         │                  │        │        │            │
        │  ┌──────┴──────┐   ┌───────▼───┐ ┌─▼──────┐ ┌──────────┐│
        │  │ HEPHAESTUS  │   │  HERMES   │ │ ARGUS  │ │ (Athena) ││
        │  │  (LXC 101)  │   │ (LXC 102) │ │(LXC103)│ └──────────┘│
        │  │  10.0.0.101 │   │10.0.0.102 │ │10.0.0. │             │
        │  │             │   │           │ │  103   │             │
        │  │  Portainer  │   │    N8N    │ │ Uptime │             │
        │  │  Twingate   │   │ Workflows │ │  Kuma  │             │
        │  │  Chisel     │   │   :5678   │ │ :3001  │             │
        │  │  :9000      │   └─────┬─────┘ └────▲───┘             │
        │  └─────────────┘         │            │                  │
        │         ▲                │            │                  │
        │         └────────────────┴────────────┘                  │
        │          Manages all     Health checks every 5 min       │
        │          containers                                      │
        │                                                           │
        └───────────────────────────────────────────────────────────┘

LEGEND:
─────►  Network flow / Connection
◄─────  Reverse proxy routing
Clients query DNS (Athena) → DNS returns Cerberus IP → 
Cerberus proxies to backend services

Monitoring Stack:
├─ N8N: Real-time alerts (Every 5 min → Discord)
└─ Uptime Kuma: Visual dashboard (Uptime %, graphs)
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
| **Olympus** | Proxmox Host | 10.0.0.200 | 8006 | Proxmox VE Hypervisor | https://10.0.0.200:8006 |


## 📸 Screenshots

### N8N Health Monitoring Workflow
<img width="1383" height="653" alt="n8n health monitor" src="https://github.com/user-attachments/assets/2745399d-0681-4db5-a802-d90c926135d7" />

*Parallel health checks with intelligent failure detection and Discord alerting*

### Discord Alert Example
<img width="1411" height="582" alt="Alert message in discord" src="https://github.com/user-attachments/assets/d132001d-7fcc-4b18-b237-d82fb44709f0" />

*Real-time notifications with service name and error details*

### Pi-hole Dashboard
<img width="1672" height="1007" alt="Athena(pihole)" src="https://github.com/user-attachments/assets/bff01864-d54b-474e-abe6-e842acd699ed" />

*Network-wide DNS filtering with custom local domain resolution*

### Portainer Container Management
<img width="1672" height="996" alt="Hephaestus(portainer)" src="https://github.com/user-attachments/assets/5ed931c7-9386-4b97-a1ef-c3b705ba9bce" />

*Docker container orchestration and monitoring*

### Uptime Kuma Dashboard

*Visual monitoring dashboard with uptime percentages and response time graphs*

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
- **5 services monitored**: All infrastructure tracked
- **HTTPS health checks**: Monitors encrypted endpoints
- **SSL-aware**: Ignores self-signed certificate warnings
- **Visual dashboard**: Real-time status in Uptime Kuma
- **99%+ uptime**: Continuous monitoring ensures availability

### Network Architecture
- **Internal DNS**: Pi-hole provides local domain resolution (*.olympus.lab)
- **Remote access**: Twingate zero-trust network access
- **Containerized**: All services run in isolated LXC containers
- **Proxmox backend**: Enterprise-grade virtualization platform


## 📚 Documentation

- [Hardware Setup](docs/01-hardware-setup.md)
- [Proxmox Installation](docs/02-proxmox-installation.md)
- [Pi-hole Deployment](docs/03-pihole-deployment.md)
- [Portainer Deployment](docs/04-portainer-deployment.md)
- [N8N Deployment](docs/05-n8n-deployment.md)
- [Twingate Setup](docs/06-twingate-setup.md)
- [Health Monitoring Workflow](docs/07-monitoring-workflow.md)
- [Troubleshooting Guide](docs/troubleshooting.md)

## 🔧 Quick Start

See [workflows/README.md](workflows/README.md) for N8N workflow setup.

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

## 📄 License

MIT License - See [LICENSE](LICENSE)

**Note:** This is a living project. Documentation and configurations are continuously updated.
