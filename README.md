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
┌─────────────────────────────────────────────┐
│        Proxmox VE (olympus.lab)            │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────────┐  ┌──────────────┐       │
│  │   Athena     │  │  Hephaestus  │       │
│  │   Pi-hole    │  │  Portainer   │       │
│  │  10.0.0.100  │  │  10.0.0.101  │       │
│  └──────────────┘  └──────────────┘       │
│                                             │
│  ┌──────────────┐                          │
│  │   Hermes     │  ← N8N Health Monitor   │
│  │     N8N      │     Every 5 minutes      │
│  │  10.0.0.102  │     Discord alerts       │
│  └──────────────┘                          │
│                                             │
│  ┌──────────────────────────────────┐     │
│  │      Twingate Connector          │     │
│  │   Zero-trust remote access       │     │
│  └──────────────────────────────────┘     │
└─────────────────────────────────────────────┘
```

## 🛡️ Services

| Service | Container | Purpose | Port |
|---------|-----------|---------|------|
| **Athena** | LXC 100 | Pi-hole DNS filtering & local DNS | 80/53 |
| **Hephaestus** | LXC 101 | Portainer container management | 9000 |
| **Hermes** | LXC 102 | N8N workflow automation & monitoring | 5678 |
| **Olympus** | Host | Proxmox VE hypervisor | 8006 |
| **Twingate** | Docker | Zero-trust network access | - |

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

## 🎓 Skills Demonstrated

- Proxmox VE virtualization
- LXC container deployment
- Docker containerization
- DNS configuration (Pi-hole)
- Workflow automation (N8N)
- Network architecture design
- Zero-trust security implementation
- Infrastructure monitoring & alerting
- Systematic troubleshooting

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

## 🙏 Acknowledgments

Built as part of AZ-800 certification preparation and homelab portfolio development.

---

**Note:** This is a living project. Documentation and configurations are continuously updated.
