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

## Time Investment

- Initial deployment: 2 hours
- DNS troubleshooting: 2 hours
- Workflow logic fixes: 1 hour
- Error handling refinement: 1 hour
- Documentation: 1 hour

**Total:** ~7 hours for production-ready monitoring infrastructure

**Value:** Portfolio-ready project, 7+ resume bullets, interview talking points
