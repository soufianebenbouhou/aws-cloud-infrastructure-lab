# Incident 03 — New instance unreachable; identical configuration to a working one

**Date:** 2026-08-27
**Layer:** Network / Routing
**Severity:** New instance unreachable from the internet. Existing instance unaffected.

---

## Symptoms

- A newly launched instance times out on SSH and on every port.
- The instance passed both status checks (2/2).
- It has a public IPv4 address assigned.
- Same VPC, same availability zone, same security group, same AMI, same key
  pair as an instance that works.
- Nothing in the EC2 console indicates a problem.
- ssh -i ~/.ssh/lab-key.pem ubuntu@<new-instance-ip>
ssh: connect to host <new-instance-ip> port 22: Operation timed out

curl -v --max-time 10 http://<new-instance-ip>:22

Trying <new-instance-ip>:22...
Connection timed out after 10002 milliseconds
curl: (28) Connection timed out after 10002 milliseconds

Symptom is identical to Incidents 01 and 02. Three different root causes across
three layers produce the same client-side output.

---

## Investigation

**1. Diff against the known-good instance.**
With a working instance in the same account, the fastest path is comparing
field by field rather than checking each subsystem in isolation. Instance type,
AMI, key pair, security group, VPC, and AZ all matched. The only difference was
the subnet.

**2. Security group ruled out.**
Both instances use `lab-sg-web`. Port 22 permits the client IP. The working
instance was reachable through the same group at the same moment.

**3. Public IP was present.**
Auto-assign was enabled on the subnet and the instance had a routable public
address. This eliminated the most common cause of an unreachable new instance.

**4. Checked the subnet's route table.**
`lab-public-c` (subnet-086c2b5298c52f63c, 10.0.3.0/24) had no explicit route
table association, so it inherited the VPC main route table,
`rtb-01966ee13b8e37c4a`.

**5. Read the main route table.**

| Destination | Target | Status | Route Origin |
|---|---|---|---|
| 10.0.0.0/16 | local | Active | CreateRouteTable |

One route. No `0.0.0.0/0`. Nothing pointing at the internet gateway. The
detail page showed `Main: Yes` and `Explicit subnet associations: –`.

---

## Commands

```bash
ssh -i ~/.ssh/lab-key.pem ubuntu@<new-instance-ip>
curl -v --max-time 10 http://<new-instance-ip>:22
hostname -I    # after recovery: 10.0.3.234, confirming the subnet
```

Console: VPC → Subnets → Route table tab; VPC → Route tables → Routes tab.

---

## Evidence

Working subnet `lab-public-a` → `lab-rt-public` (rtb-0d201d3ba7b5595df):
10.0.0.0/16 → local
0.0.0.0/0 → igw-07c7596cabc40476d

Failing subnet `lab-public-c` → main route table (rtb-01966ee13b8e37c4a):
10.0.0.0/16 → local

The subnet detail page displayed every field as normal: State Available,
IPv4 CIDR 10.0.3.0/24, Auto-assign public IPv4 Yes, Network ACL the same one
used by the working subnet, Block Public Access Off. The Route table field
showed a valid route table ID with no indication it was the default.

---

## Root cause

The subnet was created without an explicit route table association. AWS
silently associates such subnets with the VPC's main route table, which
contains only the local route.

The instance was never isolated — it remained reachable from other instances
inside the VPC, because the local route covers `10.0.0.0/16`. Only the path to
and from the internet was missing. A colleague testing from a bastion host
inside the VPC would have reported it working.

This is a failure of omission, not of misconfiguration. No rule blocked
anything. A required route simply did not exist.

---

## Fix

VPC → Route tables → `lab-rt-public` → Subnet associations → Edit subnet
associations → add `lab-public-c` → Save.

Recovery was immediate. No instance restart required; route table changes take
effect at once.

---

## Prevention

- **The warning exists, but not where the symptom sends you.** The route
  table's Subnet associations tab states it plainly: "The following subnets
  have not been explicitly associated with any route tables and are therefore
  associated with the main route table." The subnet detail page — the page
  actually opened when an instance is unreachable — shows only a route table ID.
- **Associate every subnet explicitly at creation.** Implicit association is
  invisible in list views and changes behavior for every dependent subnet if
  the main table is ever edited. Both original subnets in this lab are
  explicitly associated for that reason.
- **Keep the main route table minimal.** With only the local route, an
  unassociated subnet fails closed. Adding an IGW route to the main table would
  make every forgotten subnet publicly routable, which is a worse outcome than
  an unreachable instance.
- **"Identical to the working one" is a diagnostic, not a conclusion.** Every
  EC2 console field matched. The difference was one level down, in the subnet's
  routing. Comparing against a known-good resource is the fastest available
  technique, but the comparison has to reach past the obvious fields.
- **Observability does not follow new resources.** VPC Flow Logs were enabled
  on `lab-public-a` only, so the new subnet produced no flow records — the tool
  that solved Incident 02 was unavailable here. Monitoring configured
  per-resource silently excludes anything created later. VPC-level flow logs
  would have covered it.
- **Status checks measure the instance, not reachability.** 2/2 passing means
  the hypervisor and guest OS are healthy. It says nothing about whether
  anything can reach them.
