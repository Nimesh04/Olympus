# N8N Health Monitoring Workflow

Automated health checking for Olympus infrastructure services.

## Overview

This workflow performs parallel HTTP health checks on three critical services every 5 minutes:
- Athena (Pi-hole)
- Hephaestus (Portainer)
- Olympus (Proxmox)

If any service fails, a Discord alert is sent with detailed error information.

## Workflow Structure
```
Schedule Trigger (Every 5 min)
    ├─→ Check Athena (HTTP HEAD)
    ├─→ Check Hephaestus (HTTP HEAD)
    └─→ Check Olympus (HTTP HEAD)
         ↓ ↓ ↓ (All results merge)
    Analyze Health
    - Checks statusCodes
    - Builds failedServices array
         ↓
    Is Unhealthy? (Boolean check)
    ├─ TRUE → Alert → Discord
    └─ FALSE → Healthy (no action)
```

## Setup Instructions

### Prerequisites
- N8N instance running
- Discord webhook created
- Services to monitor accessible

### Configuration Steps

1. **Copy the example workflow**
```bash
   cp health-monitor.json.example health-monitor.json
```

2. **Replace placeholders in health-monitor.json:**
   - `{{PIHOLE_IP}}` → Your Pi-hole IP address
   - `{{PORTAINER_HOST}}` → Your Portainer hostname
   - `{{PROXMOX_HOST}}` → Your Proxmox hostname
   - `{{DISCORD_WEBHOOK_ID}}` → Your Discord webhook ID
   - `{{DISCORD_CREDENTIAL_ID}}` → Your N8N credential ID

3. **Set up Discord webhook in N8N:**
   - Settings → Credentials → Add Credential
   - Type: Discord Webhook
   - Name: Discord Bot account
   - Webhook URL: Your Discord webhook URL

4. **Import workflow:**
   - N8N UI → Workflows → Import from File
   - Select your configured `health-monitor.json`

5. **Test manually:**
   - Click "Execute Workflow"
   - Verify all checks pass

6. **Activate:**
   - Toggle "Active" switch to green
   - Workflow now runs every 5 minutes

## Error Handling

The workflow handles various error scenarios:
- `ECONNREFUSED` - Service not responding
- `EHOSTUNREACH` - Host unreachable
- `ENOTFOUND` - DNS resolution failure
- `Timeout` - Request timeout

All errors are captured and reported with service name and error details.

## Testing

Test the alerting system by stopping a service:
```bash
# Stop Portainer container
pct stop 101

# Wait for next check cycle (max 5 minutes)
# Check Discord for alert

# Restart service
pct start 101
```

## Customization

### Change check interval
Edit the Schedule Trigger node:
```json
"interval": [{"field": "minutes", "minutesInterval": 5}]
```

### Add more services
1. Add new HTTP Request node
2. Connect to Schedule Trigger
3. Connect to Analyze Health
4. Update serviceNames and serviceEmojis arrays

### Modify alert format
Edit the Alert Code node to customize Discord message format.

## Troubleshooting

**Issue: Only one service checked**
- Ensure Schedule Trigger connects to ALL check nodes (parallel execution)

**Issue: Discord not receiving alerts**
- Verify Discord webhook URL is correct
- Check N8N credentials are configured
- Test webhook URL manually with curl

**Issue: False positives**
- Increase timeout in HTTP Request nodes
- Check network connectivity
- Verify DNS resolution

## Example Alert
```
🚨 OLYMPUS ALERT 🚨

Time: 3/28/2026, 12:45:30 PM

Failed Services:
⚒️ Hephaestus (Portainer): connect ECONNREFUSED 10.0.0.101:9000

Status: 1 service(s) down
```
