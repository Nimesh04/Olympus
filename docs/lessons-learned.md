# Lessons Learned - Project Olympus

## DNS Resolution Challenges

### Problem
N8N workflow could not resolve `athena.olympus.lab` despite DNS being configured correctly with `--dns 10.0.0.100`.

### Investigation
- Verified Pi-hole had correct DNS records
- Confirmed `nslookup` worked inside container
- Found that `hephaestus.olympus.lab` and `olympus.lab` resolved successfully
- Only `athena.olympus.lab` failed consistently

### Root Cause
Node.js DNS resolution timing issue with non-standard `.lab` TLD. First DNS lookup in workflow execution failed, subsequent lookups succeeded due to DNS cache warm-up.

### Solution
Used IP address (`10.0.0.100`) for Pi-hole checks instead of domain name, eliminating DNS dependency. This is actually a best practice for monitoring critical infrastructure.

### Takeaway
For production monitoring, using IP addresses for local services is more reliable than DNS-based discovery. It eliminates a dependency and potential point of failure.

---

## Workflow Logic - Sequential vs Parallel Execution

### Problem
N8N `Analyze Health` node only received data from the last service checked, not all three services.

### Investigation
- Examined workflow structure - found checks were chained sequentially
- Each HTTP Request passed only its own result to the next node
- `Analyze Health` received only Olympus result (the final node in chain)

### Root Cause
Workflow was configured as sequential execution:
```
Schedule → Athena → Hephaestus → Olympus → Analyze Health
                                  (only this result passed forward)
```

Instead of parallel execution:
```
Schedule → Athena ────┐
        → Hephaestus ──┼→ Analyze Health (receives all 3 results)
        → Olympus ─────┘
```

### Solution
Restructured workflow so Schedule Trigger connects to ALL three check nodes simultaneously, and all three check nodes connect to Analyze Health. This enables parallel execution and result merging.

### Takeaway
Understanding N8N's execution model is critical. Sequential chaining passes single results; parallel fan-out enables result aggregation.

---

## IF Node Branch Logic

### Problem
When all services were healthy, Discord alert was triggered instead of silent success.

### Investigation
- Verified `allHealthy` flag was set correctly (true when healthy)
- Checked IF node condition (`allHealthy == false`)
- Found branches were connected backwards

### Root Cause
IF node outputs:
- Index 0 (TRUE branch) was connected to "Healthy" node
- Index 1 (FALSE branch) was connected to "Alert" node

When `allHealthy == false` evaluates to `false` (services are healthy), FALSE branch executes → triggering Alert incorrectly.

### Solution
Swapped branch connections:
- TRUE branch → Alert (unhealthy = true, trigger alert)
- FALSE branch → Healthy (unhealthy = false, no alert)

### Takeaway
Boolean logic requires careful attention to branch indexing in visual workflow tools. TRUE=index 0, FALSE=index 1.

---

## Error Object Structure Handling

### Problem
When services failed, error messages showed incorrect service names (always showed first service as failed).

### Investigation
- Found that "Continue on Error" setting returns different object structure
- Normal response: `{statusCode: 200, ...}`
- Error response: `{message: "error text", code: "ECONNREFUSED", ...}`

### Root Cause
Code assumed all responses would have `statusCode` property. Error objects had `message` and `code` instead.

### Solution
Enhanced error detection logic to check multiple properties:
```javascript
if (item.json.message && !item.json.statusCode) {
  // This is an error object
  statusCode = 0;
  errorMessage = item.json.message;
} else if (item.json.statusCode) {
  // This is a normal HTTP response
  statusCode = item.json.statusCode;
}
```

### Takeaway
When handling errors in workflow automation, check the actual object structure returned by different failure modes. Don't assume consistent schemas.

---

## Docker DNS Configuration

### Problem
Initial N8N container deployment couldn't resolve local `.olympus.lab` domains.

### Investigation
- Checked Docker's default DNS (127.0.0.11 internal resolver)
- Verified `--dns` flag was being passed correctly
- Found that even with `--dns 10.0.0.100`, resolution was unreliable

### Solution
Used `--add-host` flag to write entries directly to `/etc/hosts` file inside container:
```bash
--add-host athena.olympus.lab:10.0.0.100
--add-host hephaestus.olympus.lab:10.0.0.101
--add-host olympus.lab:10.0.0.200
```

### Takeaway
For critical service discovery in containers, `/etc/hosts` entries (via `--add-host`) are more reliable than DNS resolution, especially for non-standard TLDs.

---

## Key Principles Learned

1. **Eliminate dependencies** - IPs are more reliable than DNS for monitoring
2. **Understand execution models** - Sequential vs parallel matters
3. **Verify assumptions** - Check actual data structures, not assumed ones
4. **Test failure modes** - Error paths behave differently than success paths
5. **Document everything** - Troubleshooting notes become valuable knowledge

---

## Reverse Proxy Configuration

### Problem
Hephaestus proxy host showed in NPM UI but didn't work when accessing via browser.

### Investigation
- All services reachable directly (IP:port worked fine)
- DNS correctly resolved to Cerberus IP
- 3 out of 4 services worked through proxy
- Hephaestus returned ERR_EMPTY_RESPONSE

### Root Cause
NPM database saved proxy host entry but failed to generate Nginx config file. Config file 2.conf and later 5.conf never created on disk, so Nginx couldn't route the traffic.

### Solution
Accepted 75% success rate and used direct IP access for Hephaestus. Config regeneration bug appeared twice - not worth more troubleshooting time.

### Takeaway
Sometimes "good enough" is the right answer. Portainer accessible via direct URL is fine for rarely-used admin tool.

---

## SSL Certificate Implementation

### Problem
Services needed HTTPS but buying certificates for internal homelab is wasteful.

### Investigation
- Let's Encrypt requires public domain and DNS validation
- Internal services don't need trusted CA validation
- Self-signed certs work perfectly for encrypted traffic

### Root Cause
Not a problem - just needed proper certificate solution for internal use.

### Solution
Generated wildcard self-signed certificate covering *.olympus.lab domain. Imported into NPM and applied to all 3 working proxy hosts. Enabled Force SSL for automatic HTTP to HTTPS redirect.

### Takeaway
Self-signed certificates provide real encryption for homelab environments. Browser warnings are expected and easily bypassed for internal services.

---

## Monitoring SSL Endpoints

### Problem
After enabling HTTPS, both N8N workflow and Uptime Kuma failed with SSL certificate errors.

### Investigation
- Services worked fine in browser (after accepting cert)
- Monitoring tools rejected self-signed certificates by default
- Need to tell tools to trust our internal certificates

### Root Cause
Security feature - monitoring tools don't trust unknown certificate authorities by default.

### Solution
**Uptime Kuma:** Enabled "Ignore TLS/SSL error" checkbox in monitor settings.
**N8N:** Enabled SSL ignore option in HTTP Request node settings.

### Takeaway
When monitoring HTTPS services with self-signed certs, explicitly configure tools to ignore certificate validation. Security trade-off acceptable for internal monitoring.

---

## Docker Bridge Network Limitations

### Problem
NPM container could reach some LXC containers but behavior was inconsistent.

### Investigation
- NPM in default Docker bridge network (172.17.0.x)
- LXC containers on Proxmox network (10.0.0.x)
- Bridge mode worked for 3 services but not Hephaestus

### Root Cause
Docker bridge networking can have routing issues reaching external container IPs. Host networking mode would provide direct network access.

### Solution
Didn't fix - accepted working state. Could redeploy NPM with --network host for full LXC connectivity.

### Takeaway
Docker bridge vs host networking matters for inter-container communication. Host mode more reliable for proxying to external services.



---

## Backup & Disaster Recovery Implementation

### Problem
No backup system in place meant catastrophic data loss risk from hardware failure, accidental deletion, or corruption.

### Investigation
- Mapped all 6 containers and their criticality levels
- Calculated storage requirements: 4.6GB per backup set
- Determined acceptable RTO (2 hours) and RPO (24 hours)

### Root Cause
Infrastructure without backups is a single point of failure. Every SysAdmin role requires B&DR experience.

### Solution
Implemented automated Proxmox backup system with:
- Daily full backups at 2 AM using vzdump
- ZSTD compression reducing backup size by ~60%
- 7-day retention (32GB maximum storage usage)
- Python-based monitoring via N8N (daily 8 AM verification)
- Discord alerts on backup failure

### Takeaway
Backups are worthless until proven recoverable. Tested full container restoration achieving 2-minute RTO (60x better than 2-hour target). Automated monitoring ensures backup failures are detected within 6 hours.

---

## Backup Monitoring Challenges

### Problem
N8N SSH node returned file list instead of count when using standard bash pipes.

### Investigation
- Direct shell: `ls | wc -l` returned `6` (correct)
- N8N SSH node: Same command returned file list + `0` (incorrect)
- Issue: Non-interactive shell behavior in N8N SSH execution

### Root Cause
N8N's SSH node executes commands in non-interactive mode where pipe operators may not behave as expected. Shell interpretation differs from standard interactive sessions.

### Solution
Used Python one-liner instead of bash pipes:
```python
python3 -c "import glob; print(len(glob.glob('/var/lib/vz/dump/vzdump-lxc-*.tar.zst')))"
```

### Takeaway
When automating commands across different execution environments (interactive shell vs SSH vs cron), prefer language-specific solutions (Python, Perl) over bash pipes for predictable behavior. Python's glob module provides consistent results regardless of shell context.

---

## Time Investment

- Backup system design: 30 minutes
- Proxmox backup configuration: 15 minutes
- Disaster recovery testing: 10 minutes
- Monitoring workflow setup: 45 minutes
- Documentation: 20 minutes


