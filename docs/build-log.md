# Build Log

AWS Cloud Infrastructure Lab — chronological record of what was built,
what it cost, and what broke.

Account ID redacted. Region: us-west-2 (Oregon).
Note: all AWS console and billing timestamps are UTC.

---

## Phase 0 — Account and guardrails (2026-08-26)

### Region
- us-west-2 (Oregon). Console default for this account; lower latency from
  Southern California; pricing identical to us-east-2 for everything in this lab.
- Pinned as default in console settings to prevent split-region builds.

### Cost controls
- Budget `lab-zero-spend` — $1.00/month, COST type, ACTUAL comparison,
  $0.01 ABSOLUTE_VALUE threshold, email notification.
- No `CostTypes` block in the budget definition, so API defaults apply:
  `IncludeCredit=true`. The budget therefore measures NET cost.
- Implication: it stays silent while the $100 promotional credit absorbs
  charges. It fires when the card is about to be charged, not when credits
  start draining.
- TODO after EC2 has run 48h: second budget, monthly, credits EXCLUDED,
  ~$15 threshold, as a gross burn-rate tripwire.
- Manual check: Billing > Credits, weekly. No alarm replaces the fuel gauge.

### Root account security
- MFA enabled: `root-mfa` (virtual, phone authenticator)
- MFA backup: `root-mfa-backup` (passkey, Touch ID, iCloud-synced)
- Root access keys: 0. None will be created. CLI access in Phase 4 will use
  an IAM principal.
- Recovery contacts verified.

---

## Incident 00 — Cannot register second MFA device

**Symptoms**
"Error registering security key — To complete this action, please ensure that
you are authenticated with an MFA device that is enabled for this user."
Raised during passkey registration.

**Investigation**
Error text implicates the passkey. The passkey was not the cause. The console
session had been established before any MFA device existed on the account, so
the session token carried no MFA context.

**Root cause**
AWS requires an MFA-authenticated session to register additional MFA devices.
Any device type would have failed identically.

**Fix**
Sign out. Sign back in, completing the MFA challenge with the existing device.
Retry registration in the new session.

**Prevention**
Register all root MFA devices in one MFA-authenticated session, or expect one
re-authentication cycle after the first device is added.

---

## Phase 1 — Network foundation

### VPC — 2026-08-26
- Name:    lab-vpc
- ID:      vpc-03c04264c547d4e52
- CIDR:    10.0.0.0/16
- Tenancy: default
- Default VPC: No
- IPv6:    none
- DNS resolution: Enabled (on by default)
- DNS hostnames:  Enabled (set MANUALLY — off by default on non-default VPCs)
- Block Public Access: Off (intentional — this lab exposes a public API)

Auto-created by AWS:
- main route table  rtb-01966ee13b8e37c4a
- main network ACL  acl-000b0badb87d6dcc3
- DHCP option set   dopt-0d26f22e4b9b0ace1

**Decisions**
- Chose "VPC only" over "VPC and more". The wizard provisions a NAT Gateway
  (~$32/mo). Combined with a net-cost budget, that charge would have been
  invisible until the credits were gone.
- /16 CIDR for clean /24 subnet boundaries.
- No IPv6 — avoids maintaining a second addressing scheme across route tables
  and security group rules.
- DNS hostnames enabled deliberately: without it, EC2 instances receive no
  public DNS name, and the planned DNS failure scenario has no working
  baseline to break.

**Note for later**
Subnets with no explicit route table association inherit the main route table,
which has only the local route. Such a subnet appears correctly configured in
list views and is unreachable from the internet. Candidate break/fix scenario.

**Cost:** $0. VPCs, subnets, IGWs, route tables, security groups and NACLs
are not billed.

### Subnets — 2026-08-26

| Name          | ID                       | CIDR        | AZ         | Usable IPs |
|---------------|--------------------------|-------------|------------|------------|
| lab-public-a  | subnet-0ed69202000fc6f62 | 10.0.1.0/24 | us-west-2a | 251        |
| lab-private-b | subnet-071d8d711088b07bd | 10.0.2.0/24 | us-west-2b | 251        |

**Notes**
- Neither subnet is "public" or "private" at creation. Both inherit the main
  route table (rtb-01966ee13b8e37c4a), which contains only the local route.
  The distinction is created by routing in the next step, not by any subnet
  setting.
- Separate AZs: physically distinct facilities. No cost, and an AZ-scoped
  failure affects only one subnet.
- AWS reserves 5 addresses per subnet (network, VPC router, DNS, reserved,
  broadcast), so a /24 yields 251 usable, not 256.
- Default VPC (vpc-05bb676a42ec639af) holds four unnamed 172.31.x.x/20
  subnets. Unused by this lab, left in place.

**Cost:** $0.

### Internet Gateway — 2026-08-26
- Name:  lab-igw
- ID:    igw-07c7596cabc40476d
- State: Attached to vpc-03c04264c547d4e52

**Notes**
- Creation and attachment are separate operations. A detached IGW is inert.
- Attachment alone does not create connectivity. At this point the IGW is
  attached and both subnets remain unreachable, because no route table
  contains a route pointing to it. Routing is what makes a subnet public.
- IGWs are free: horizontally scaled, redundant, no hourly charge, no data
  processing fee. NAT Gateways are the opposite on all three counts
  (~$0.045/hr plus per-GB processing), which is why the "VPC and more"
  wizard was avoided.

**Cost:** $0.

### Route tables — 2026-08-26

| Name           | ID                    | Associated subnet                     | Routes |
|----------------|-----------------------|---------------------------------------|--------|
| lab-rt-public  | rtb-0d201d3ba7b5595df | subnet-0ed69202000fc6f62 (public-a)   | 0.0.0.0/0 → igw-07c7596cabc40476d; 10.0.0.0/16 → local |
| lab-rt-private | rtb-09e671874c7873055 | subnet-071d8d711088b07bd (private-b)  | 10.0.0.0/16 → local |
| (main)         | rtb-01966ee13b8e37c4a | none (no explicit associations)       | 10.0.0.0/16 → local |

**Notes**
- This step, not the IGW, is what makes a subnet public. The IGW was attached
  in the previous step and nothing was reachable until 0.0.0.0/0 pointed at it.
- Longest-prefix-match: traffic to 10.0.x.x matches the /16 local route and
  stays in the VPC; everything else falls through to the /0 and exits via the
  gateway. The two routes do not conflict.
- The local route is created by AWS (Route Origin: CreateRouteTable) and
  cannot be deleted. The default route shows CreateRoute — useful for
  distinguishing operator changes from AWS defaults when auditing.
- lab-private-b was explicitly associated with lab-rt-private rather than left
  to inherit the main table. Implicit association is invisible in most views
  and changes behavior for every dependent subnet if the main table is edited.
- Main route table now has zero explicit associations. Any new subnet created
  without an association will inherit it and be unreachable while appearing
  correctly configured. Candidate break/fix scenario.

**Cost:** $0.
