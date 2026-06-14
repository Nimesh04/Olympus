# Project Olympus

Self-hosted homelab infrastructure running on a 2-node Proxmox cluster. Greek mythology naming throughout, because it made sense early on and now we're committed.

![Status](https://img.shields.io/badge/status-active-success)
![Monitoring](https://img.shields.io/badge/monitoring-automated-blue)
![Platform](https://img.shields.io/badge/platform-Proxmox%20VE-orange)
![Cluster](https://img.shields.io/badge/cluster-2%20nodes-purple)
![Services](https://img.shields.io/badge/services-7%20running-yellow)

## What this is

Started as a single ThinkPad running Proxmox with a handful of LXC containers doing useful things. Then a Dell Precision T5810 with 64GB of RAM showed up and it turned into a proper cluster.

The X1 Carbon (homelab) handles all the containerized services — DNS, reverse proxy, monitoring, file sync, and automation. The T5810 (Hyperion) joined the cluster and now hosts the Windows Server and Active Directory lab for AZ-800 certification prep. An old Acer laptop sitting unused became the quorum device.

Three nodes. Three-way quorum. Split-brain protection. One management interface.

## Cluster Olympus

| Node | Hardware | Role | IP |
|------|----------|------|----|
| homelab | ThinkPad X1 Carbon Gen 5 | Primary node · 7 LXC containers | 10.0.0.200 |
| Hyperion | Dell Precision T5810 · 64GB RAM | Secondary node · Windows VM lab | 10.0.0.201 |
| quorum | Acer laptop | QDevice · corosync-qnetd | 10.0.0.202 |

If the two main nodes lose communication, the Acer acts as tiebreaker — one node wins, the other shuts down cleanly. No data corruption, no split-brain conflict.

Getting the cluster working involved a few mistakes: editing Netplan without sudo and wondering why nothing saved, connecting to the wrong IP during cluster join and getting hostname errors, trying to install the wrong corosync package on the wrong machine. All fixed. All understood.

## Services

All running as LXC containers on the homelab node.

| Service | Container | IP | Purpose |
|---------|-----------|-----|---------|
| Athena | LXC 100 | 10.0.0.100 | Pi-hole DNS + ad blocking |
| Hephaestus | LXC 101 | 10.0.0.101 | Portainer container management |
| Hermes | LXC 102 | 10.0.0.102 | N8N workflow automation |
| Argus | LXC 103 | 10.0.0.103 | Uptime Kuma monitoring |
| Cerberus | LXC 104 | 10.0.0.104 | Nginx Proxy Manager + SSL |
| Artemis | LXC 105 | 10.0.0.105 | Nextcloud + PostgreSQL |
| Apollo | LXC 106 | 10.0.0.106 | Prometheus + Grafana + Alertmanager |

## Active Directory Lab (Hyperion)

Running on Hyperion as part of AZ-800 (Administering Windows Server Hybrid Core Infrastructure) prep.

| VM | IP | Role |
|----|-----|------|
| DC01 | 10.0.0.107 | Domain controller · Windows Server 2022 |

Domain: `olympuscorp.local`

OU structure mirrors a real company layout with IT, Finance, and HR departments. Entra Connect configured for hybrid identity sync to Azure — password hash sync and password writeback enabled. DC02 (secondary domain controller) in progress.

## Architecture

![Cluster Olympus Architecture](./docs/architecture.svg)

Traffic flows from the home network into the homelab node. Cerberus handles SSL termination and reverse proxying for all internal services. Athena resolves `*.olympus.lab` hostnames. Remote access runs through Twingate — no open ports, no port forwarding.

Apollo (Prometheus + Grafana + Alertmanager) monitors all seven containers via Node Exporter. Alerts fire to Discord when something breaks. Hermes (N8N) also runs a daily backup check at 8 AM and sends a Discord alert if expected backup files are missing.

The two Proxmox nodes and the Acer QDevice maintain cluster quorum via corosync. Token timeout is set to 5000ms to avoid false-positive node failures under load.

## Monitoring

Alert rules running on Apollo:

| Alert | Fires when | Severity |
|-------|-----------|----------|
| ContainerDown | Target unreachable > 1 min | critical |
| HighCPUUsage | CPU > 80% for 2 min | warning |
| HighMemoryUsage | Memory > 85% for 2 min | warning |
| LowDiskSpace | Disk < 15% for 5 min | warning |
| CriticalDiskSpace | Disk < 5% for 1 min | critical |

Grafana runs the Node Exporter Full dashboard (ID: 1860). All alerts route to a dedicated Discord channel.

## Backup and DR

Daily `vzdump` at 2:00 AM. ZSTD compression. 7-day retention.

LXC container backups go to the homelab's local storage. VM backups for DC01 go to a dedicated backup volume on Hyperion — a 200GB logical volume on the `vmdata` VG, mounted at `/mnt/backups`.

Tested recovery time: **2 minutes** (validated April 11, 2026).

Restore a container:

```bash
pct stop <CTID>
pct restore <CTID> /var/lib/vz/dump/vzdump-lxc-<CTID>-<timestamp>.tar.zst
pct start <CTID>
```

## Screenshots

### N8N Health Monitoring Workflow (Hermes)
<img width="1531" height="811" alt="N8N monitoring workflow" src="https://github.com/user-attachments/assets/6d2e4588-9a97-4fae-8d2c-ac17cd8b932d" />

### Discord Alert
<img width="1411" height="582" alt="Discord alert example" src="https://github.com/user-attachments/assets/d132001d-7fcc-4b18-b237-d82fb44709f0" />

### Pi-hole (Athena)
<img width="1672" height="1007" alt="Pi-hole dashboard" src="https://github.com/user-attachments/assets/bff01864-d54b-474e-abe6-e842acd699ed" />

### Portainer (Hephaestus)
<img width="1672" height="996" alt="Portainer dashboard" src="https://github.com/user-attachments/assets/5ed931c7-9386-4b97-a1ef-c3b705ba9bce" />

### Uptime Kuma (Argus)
<img width="1671" height="872" alt="Uptime Kuma dashboard" src="https://github.com/user-attachments/assets/ea876a72-535c-4f1b-b24b-a4e89f4cba64" />

### Nginx Proxy Manager (Cerberus)
<img width="1667" height="960" alt="Nginx Proxy Manager" src="https://github.com/user-attachments/assets/5a7b079d-af9e-4712-9371-3e04e6c6607d" />

### Nextcloud (Artemis)
<img width="1675" height="928" alt="Nextcloud dashboard" src="https://github.com/user-attachments/assets/44486203-43f2-4263-b023-e58bf23b81da" />

### Prometheus + Grafana (Apollo)
<img width="1911" height="972" alt="Grafana dashboard" src="https://github.com/user-attachments/assets/cc9c4870-b9bd-4db2-b142-4c59862d4b24" />
<img width="1912" height="966" alt="Prometheus targets" src="https://github.com/user-attachments/assets/8c2c2371-2df5-48cb-80a3-06da2eddcd08" />

## Specs

### homelab (X1 Carbon)

| Component | Detail |
|-----------|--------|
| Hardware | ThinkPad X1 Carbon Gen 5 |
| CPU | Intel Core i5-7300U |
| RAM | 16GB |
| Storage | 500GB SSD |
| Hypervisor | Proxmox VE 9.x |
| Container OS | Debian 12 / Ubuntu 24.04 |

### Hyperion (T5810)

| Component | Detail |
|-----------|--------|
| Hardware | Dell Precision T5810 |
| CPU | Intel Xeon E5-2630 v4 |
| RAM | 64GB DDR4 |
| Storage | 240GB SSD (OS + VMs) · 1TB HDD (bulk + backups) |
| Hypervisor | Proxmox VE 9.x |
| VM OS | Windows Server 2022 |

## Security notice

All credentials, webhook URLs, and real IP ranges have been replaced with placeholders. Don't commit real secrets.

## Lessons learned

See [docs/lessons-learned.md](docs/lessons-learned.md) for things that broke and why.