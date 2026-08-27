# Incident 01 — API unreachable from the internet, SSH unaffected

**Date:** 2026-08-27
**Layer:** Network / Security group
**Severity:** Full outage of the public API. No server-side error signal.

---

## Symptoms

- `GET /health` from outside the VPC hangs and times out after 10s.
- SSH to the same host on port 22 succeeds normally.
- No errors in the application journal.
- No CloudWatch alarms. Instance status checks 2/2 passing.
- The failure is visible only from the client side.
- curl -v --max-time 10 http://<public-ip>:5000/health

Trying <public-ip>:5000...
connect to <public-ip> port 5000 failed: Operation timed out
curl: (28) Failed to connect to <public-ip> port 5000

---

## Investigation

**1. Timeout, not refusal.**
The connection timed out rather than returning `Connection refused`. A refusal
means a TCP RST came back — the packet reached the host and nothing was
listening. A timeout means the packet was silently dropped. Security groups
drop; they do not send RST. This pointed at a network filter, not the service.

**2. SSH still worked.**
That single fact eliminated instance state, the public IP, the internet
gateway, the route table, and subnet configuration. The host was reachable.
The failure was specific to port 5000.

**3. The service was healthy.**
`systemctl status lab-api` showed active (running). `curl localhost:5000/health`
from inside the instance returned 200. The application was not at fault.

**4. The bind address was correct.**
`ss -tlnp | grep 5000` showed `0.0.0.0:5000`, not `127.0.0.1:5000`. This ruled
out a loopback bind, which produces identical external symptoms with a
completely different root cause and fix.

**5. The application log was empty of connection attempts.**
Absence of log entries was itself evidence: the requests never arrived. A
service-side fault would have generated entries. Silence during a reported
outage indicates the problem is upstream of the application.

**6. Read the inbound rules in the console.**
The instance role has no `ec2:DescribeSecurityGroups` permission by design, so
this step could not be done from the CLI on the host.

---

## Commands

```bash
# From the client
curl -v --max-time 10 http://<public-ip>:5000/health
ssh -i ~/.ssh/lab-key.pem ubuntu@<public-ip>

# On the instance
sudo systemctl status lab-api --no-pager
curl -s localhost:5000/health
sudo ss -tlnp | grep 5000
sudo journalctl -u lab-api -n 30 --no-pager
```

---

## Evidence

Service healthy and listening on all interfaces:
Active: active (running)
LISTEN 0 2048 0.0.0.0:5000 0.0.0.0:* users:(("gunicorn",pid=...))
{"status":"ok","time":"..."}

Inbound rules on lab-sg-web at time of failure:

| Type       | Port | Source          | Result             |
|------------|------|-----------------|--------------------|
| SSH        | 22   | <my-ip>/32      | working            |
| HTTP       | 80   | 0.0.0.0/0       | n/a                |
| Custom TCP | 5000 | **10.0.0.0/16** | **blocking**       |

CloudTrail `ModifySecurityGroupRules` event confirmed the change, the principal
that made it, and the timestamp.

---

## Root cause

The inbound rule for port 5000 had its source set to `10.0.0.0/16` — the VPC
CIDR — instead of `0.0.0.0/0`. Traffic originating inside the VPC was
permitted; traffic from the internet was dropped.

SSH was unaffected because its rule was never modified, which is why the host
appeared healthy by every other measure.

---

## Fix

EC2 → Security Groups → `lab-sg-web` → Edit inbound rules → port 5000 source
back to `0.0.0.0/0` → Save.

Recovery was immediate. Security group changes apply to new connections at
once, with no restart or propagation delay.

---

## Prevention

- **Read the failure mode before the config.** Timeout vs. `Connection refused`
  splits the problem in half before anything is opened in the console. Every
  subsequent step should narrow from there.
- **A working SSH session is a diagnostic asset.** It proves routing, gateway,
  and instance health in one command, and isolates the failure to a port.
- **This class of failure produces no server-side signal.** Application logs,
  CloudWatch metrics, and status checks are all clean. Detection depends on
  external synthetic checks, not on server-side monitoring. A canary hitting
  `/health` from outside the VPC would have caught it; nothing running on the
  host would have.
- **Use rule descriptions.** Every rule in this security group carries a
  description of its purpose, so an incorrect source is visible when reading
  the rules rather than requiring inference.
- **CloudTrail Event History is free and retains 90 days of management events.**
  `ModifySecurityGroupRules`, `AuthorizeSecurityGroupIngress`, and
  `RevokeSecurityGroupIngress` are all recorded with principal and timestamp.
  This is the first place to look when configuration changed and no one
  remembers changing it.
- **Optional:** VPC Flow Logs would have recorded REJECT entries for the
  dropped packets, giving a server-side signal for exactly this failure. Not
  enabled here — it bills CloudWatch Logs ingestion — but it is the standard
  answer for making silent drops visible.
  
