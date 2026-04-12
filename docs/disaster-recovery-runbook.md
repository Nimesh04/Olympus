# Disaster Recovery Runbook - Project Olympus

## Quick Reference

Backup Location: `/var/lib/vz/dump/` on Proxmox host  
Backup Schedule: Daily at 2:00 AM  
Retention: 7 days  
RTO Target: 2 hours  
RPO Target: 24 hours  

---

## Common Scenarios

### Scenario 1: Accidental Container Deletion

Symptoms:
- Container missing from Proxmox UI
- `pct list` doesn't show container
- Services unreachable

Recovery Steps:

1. Identify backup file:
```bash
ls -lt /var/lib/vz/dump/vzdump-lxc-<CTID>-* | head -1
```

2. Restore container:
```bash
pct restore <CTID> /var/lib/vz/dump/vzdump-lxc-<CTID>-<timestamp>.tar.zst
```

3. Start container:
```bash
pct start <CTID>
```

4. Verify services:
- Check container status: `pct status <CTID>`
- Test network: `ping 10.0.0.<IP>`
- Access web UI (if applicable)

Expected Recovery Time: 2-5 minutes

---

### Scenario 2: Corrupted Container Filesystem

Symptoms:
- Container won't start
- "filesystem error" in logs
- Services fail to load

Recovery Steps:

1. Stop container:
```bash
pct stop <CTID>
```

2. Rename corrupted container:
```bash
mv /var/lib/vz/images/<CTID> /var/lib/vz/images/<CTID>.corrupted
```

3. Restore from backup:
```bash
pct restore <CTID> /var/lib/vz/dump/vzdump-lxc-<CTID>-<timestamp>.tar.zst
```

4. Start and verify:
```bash
pct start <CTID>
pct status <CTID>
```

Expected Recovery Time: 3-6 minutes

---

### Scenario 3: Proxmox Host Failure (Hardware)

Symptoms:
- Proxmox host won't boot
- Hardware failure detected
- Complete system loss

Recovery Steps:

Prerequisites:
- New/replacement hardware
- Proxmox VE installation media
- Backup files (external drive or network storage)

1. Install Proxmox on new hardware
2. Restore network configuration (10.0.0.200, same subnet)
3. Copy backup files to `/var/lib/vz/dump/`
4. Restore all containers in order:

```bash
# Critical infrastructure first
pct restore 100 /var/lib/vz/dump/vzdump-lxc-100-*.tar.zst  # Athena (DNS)
pct restore 104 /var/lib/vz/dump/vzdump-lxc-104-*.tar.zst  # Cerberus (Proxy)

# Then supporting services
pct restore 102 /var/lib/vz/dump/vzdump-lxc-102-*.tar.zst  # Hermes
pct restore 103 /var/lib/vz/dump/vzdump-lxc-103-*.tar.zst  # Argus
pct restore 105 /var/lib/vz/dump/vzdump-lxc-105-*.tar.zst  # Artemis
pct restore 101 /var/lib/vz/dump/vzdump-lxc-101-*.tar.zst  # Hephaestus
```

5. Start all containers:
```bash
for i in 100 101 102 103 104 105; do pct start $i; done
```

6. Verify all services

Expected Recovery Time: 1-2 hours (including Proxmox install)

---

## Backup Verification

### Daily Automated Check

N8N workflow runs at 8:00 AM:
- Validates 6 backup files exist
- Alerts via Discord if any missing
- Python command: `python3 -c "import glob; print(len(glob.glob('/var/lib/vz/dump/vzdump-lxc-*.tar.zst')))"`

### Manual Verification

```bash
# List all backups
ls -lh /var/lib/vz/dump/*.tar.zst

# Check backup logs for errors
grep -i error /var/lib/vz/dump/*.log

# Verify backup sizes (should be >100MB each)
du -sh /var/lib/vz/dump/*.tar.zst
```

---

## Container Priority Order

Recovery order during major outage:

1. Athena (100) - DNS required for everything else
2. Cerberus (104) - Reverse proxy for HTTPS access
3. Hermes (102) - Automation and monitoring workflows
4. Argus (103) - Uptime monitoring
5. Artemis (105) - File storage
6. Hephaestus (101) - Container management (can rebuild manually if needed)

---

## Contact Information

Proxmox Access:
- URL: https://10.0.0.200:8006
- User: root
- SSH: `ssh root@10.0.0.200`

Discord Alerts:
- Backup failures posted to your configured Discord channel

Documentation:
- GitHub: Your repository URL
- Local: D:\Certifications\Olympus\

---

## Testing Schedule

Monthly: Test restore of one random container  
Quarterly: Full disaster recovery simulation  
Annually: Restore all containers to verify complete system recovery

Last Test: April 11, 2026 - LXC 101 (Hephaestus) - SUCCESS (2 min RTO)
