# Incident 02 — Total subnet outage: request accepted, reply dropped

**Date:** 2026-08-27
**Layer:** Network / Network ACL
**Severity:** Complete loss of inbound access on all ports. SSH also lost, so
recovery had to happen entirely from the console.

---

## Symptoms

- `GET /health` from outside the VPC times out after 10s.
- SSH on port 22 also times out. Both ports affected.
- An already-established SSH session died mid-command:
  `Read from remote host: Operation timed out` / `client_loop: send disconnect: Broken pipe`
- No application errors. Service healthy. Status checks 2/2.
- curl -v --max-time 10 http://<public-ip>:5000/health

Trying <public-ip>:5000...
Connection timed out after 10003 milliseconds
curl: (28) Connection timed out after 10003 milliseconds

ssh: connect to host <public-ip> port 22: Operation timed out

Timeline:
22:12:09 last successful request
22:12:18 change applied
22:12:20 established SSH session dropped mid-session
22:21:35 recovery confirmed, HTTP 200

---

## Investigation

**1. Both ports failed, not just one.**
This is the first divergence from Incident 01. There, SSH kept working and
isolated the fault to a single port — pointing at a security group rule. Here
22 and 5000 both timed out, which indicates something subnet-wide: a NACL or a
route, not a per-port rule.

**2. An established connection was killed.**
The live SSH session did not survive. Packet filters evaluate every packet
independently; they do not grandfather existing sessions. A filter change is
not "new connections only" — it takes effect immediately on traffic in flight.

**3. Security group was correct.**
Port 5000 source `0.0.0.0/0`, port 22 source `<my-ip>/32`. Unchanged from the
known-good baseline.

**4. Routing was correct.**
`lab-rt-public` had `0.0.0.0/0 → igw-07c7596cabc40476d`, Active, with
`lab-public-a` associated.

**5. VPC Flow Logs identified the cause.**
Query run in CloudWatch Logs Insights against `/lab/vpc-flow-logs`.

---

## Commands

Client:
```bash
curl -v --max-time 10 http://<public-ip>:5000/health
ssh -i ~/.ssh/lab-key.pem ubuntu@<public-ip>
```

CloudWatch Logs Insights:
fields @timestamp, srcAddr, dstAddr, srcPort, dstPort, action
| filter dstPort = 5000 or srcPort = 5000
| sort @timestamp desc
| limit 50

---

## Evidence

Flow log records across the fault boundary:

| @timestamp | srcAddr | dstAddr | srcPort | dstPort | action |
|---|---|---|---|---|---|
| 22:21:36 | 10.0.1.65 | 47.41.17.32 | 5000 | 49685 | ACCEPT |
| 22:21:36 | 47.41.17.32 | 10.0.1.65 | 49685 | 5000 | ACCEPT |
| 22:20:36 | 10.0.1.65 | 47.41.17.32 | 5000 | 49680 | **REJECT** |
| 22:20:36 | 47.41.17.32 | 10.0.1.65 | 49680 | 5000 | **ACCEPT** |
| 22:19:35 | 10.0.1.65 | 47.41.17.32 | 5000 | 49676 | **REJECT** |
| 22:19:05 | 10.0.1.65 | 47.41.17.32 | 5000 | 49676 | **REJECT** |
| 22:19:05 | 47.41.17.32 | 10.0.1.65 | 49676 | 5000 | **ACCEPT** |

**The inbound packet is ACCEPT. The outbound reply on the same flow is REJECT.**

That pattern is only possible with a stateless filter. Security groups track
connection state — once an inbound connection is allowed, its return traffic is
permitted automatically, and no configuration makes a security group accept a
request and then reject its own reply. Independent evaluation per direction
means NACL.

Repeated REJECTs on port 49676 at 22:19:05 and 22:19:35 are TCP
retransmissions: the client received nothing, retried, and eventually gave up
at the 10s timeout.

NACL outbound rules at time of failure:

| Rule | Type | Port range | Destination | Allow/Deny |
|---|---|---|---|---|
| 50 | Custom TCP | 1024-65535 | 0.0.0.0/0 | **DENY** |
| 100 | All traffic | All | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | 0.0.0.0/0 | DENY |

---

## Root cause

An outbound DENY rule numbered 50 blocked TCP destination ports 1024-65535 on
the main network ACL.

Two mechanics made this effective:

**Ephemeral ports, not the service port.** The reply leaves *from* port 5000
*to* the client's ephemeral port. The flow logs show 49676, 49680, 49685 —
macOS allocates from 49152-65535. A deny on outbound port 5000 would have had
no effect, because 5000 is the source port on the way out.

**Rule ordering.** NACLs evaluate in ascending rule number and stop at the
first match. Rule 100 (Allow all) was still present and still correct — it was
simply never reached. Adding a NACL rule does not replace anything; it inserts
ahead of existing rules.

---

## Fix

VPC → Network ACLs → main NACL → Outbound rules → Edit → remove rule 50 → Save.

Recovery was immediate: HTTP 200 at 22:21:35, SSH at 22:21:38.

---

## Secondary incident during recovery

Removing rule 50 in the console also removed rule 100, leaving the outbound
ruleset with only the implicit `*` DENY. The subnet remained fully blocked,
now for a different reason than the original fault.

The `*` rule is always present, cannot be removed, and denies anything no rule
explicitly allows. A NACL with no ALLOW rules is not "unconfigured" — it is a
complete block.

Fixed by re-adding rule 100 (All traffic, 0.0.0.0/0, ALLOW).

Worth recording because a recovery action that causes a second outage is
common in practice and rarely written down. The console gives no warning when
the last ALLOW rule is removed.

---

## Prevention

- **Two ports failing instead of one narrows the layer immediately.** A
  single-port failure suggests a security group; all ports suggests NACL or
  routing. That distinction is free and should be the first question asked.
- **Stateful vs stateless is the core concept here.** Security groups return
  traffic automatically; NACLs require an explicit rule in each direction. Any
  NACL change to one direction needs the other direction checked.
- **Ephemeral port ranges matter in NACL rules.** Return traffic is addressed
  to the client's ephemeral port, not the service port. Linux clients typically
  use 32768-60999; macOS and Windows use 49152-65535. A NACL that allows
  inbound on a service port but not outbound on the ephemeral range will drop
  every reply.
- **Flow logs are the only way to see this.** No host-side log records a
  packet that the VPC dropped. Application logs, journald, and CloudWatch
  metrics were all clean throughout. Without flow logs the diagnosis is
  guesswork.
- **Filter changes kill live connections.** They do not apply to new
  connections only. Losing SSH during a NACL edit is a realistic way to lock
  yourself out of an instance.
- **Prefer security groups; use NACLs sparingly.** Security groups are
  stateful, instance-scoped, and much harder to misconfigure this way. NACLs
  are a subnet-wide blunt instrument and should be reserved for coarse denies.
- **Cost note:** flow logs bill CloudWatch Logs ingestion (~$0.50/GB) and
  retention was set to 7 days at creation. Enabling them permanently on a busy
  VPC is a real expense; enabling them during an investigation is cheap.
