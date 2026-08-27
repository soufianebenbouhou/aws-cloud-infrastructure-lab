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

### Security group — 2026-08-26
- Name: lab-sg-web
- ID:   sg-0d9c0f37446e20b71
- VPC:  vpc-03c04264c547d4e52

**Inbound (3)**

| Rule ID                 | Type       | Protocol | Port | Source      | Purpose        |
|-------------------------|------------|----------|------|-------------|----------------|
| sgr-069620a57c0d6206f   | SSH        | TCP      | 22   | <my-ip>/32  | SSH from home  |
| sgr-075f59cfabe598e2d   | HTTP       | TCP      | 80   | 0.0.0.0/0   | HTTP           |
| sgr-07b9999c3d4cd5536   | Custom TCP | TCP      | 5000 | 0.0.0.0/0   | Flask API      |

**Outbound (1)** — default: all traffic to 0.0.0.0/0. Required for package
installs and S3 access.

**Notes**
- Security groups are STATEFUL. An inbound allow implies the return traffic is
  permitted outbound; no matching outbound rule is needed. NACLs are stateless
  and require rules in both directions. That difference is the source of most
  hard-to-diagnose NACL failures and is a planned break/fix scenario.
- SSH source is pinned to a single /32 via "My IP". Residential IPs rotate on
  router reboot or network change. Symptom when it happens: SSH connection
  timeout with no change on the instance. Fix: re-select "My IP" in the rule.
  Planned break/fix scenario.
- Port 5000 is open to 0.0.0.0/0 deliberately, so Postman can reach the API
  from anywhere and so TLS/DNS scenarios can be exercised against it. A real
  deployment would place this behind a load balancer or restrict the source.
- Per-rule IDs (sgr-*) allow modifying a single rule via CLI rather than
  replacing the whole ruleset. Useful for scripted break/fix.
- Actual SSH source IP kept out of this repo.

**Cost:** $0.

---

## Phase 1 complete — 2026-08-26

| Resource       | ID                       |
|----------------|--------------------------|
| VPC            | vpc-03c04264c547d4e52    |
| Public subnet  | subnet-0ed69202000fc6f62 |
| Private subnet | subnet-071d8d711088b07bd |
| Internet GW    | igw-07c7596cabc40476d    |
| Public RT      | rtb-0d201d3ba7b5595df    |
| Private RT     | rtb-09e671874c7873055    |
| Security group | sg-0d9c0f37446e20b71     |

Running cost: $0.00. Credits used: $0.00 of $100.

---

## Phase 2 — Compute

### Key pair — 2026-08-26
- Name: lab-key (RSA, .pem)
- Stored at ~/.ssh/lab-key.pem, chmod 400
- Downloadable exactly once. No recovery path — losing it means terminating
  and relaunching any instance that uses it.
- Covered by .gitignore (*.pem). Never committed.
- chmod 400 is required, not advisory. SSH refuses a private key readable by
  anyone else and exits with UNPROTECTED PRIVATE KEY FILE. Browser downloads
  land at 644.

### EC2 instance — 2026-08-27
- Name:        lab-web-01
- Instance ID: i-0a5d4684f2c4f3008
- Type:        t3.micro (2 vCPU, 908 MiB usable RAM), hvm, tenancy default
- AMI:         Ubuntu Server 26.04 LTS (x86_64), kernel 7.0.0-1006-aws
- Storage:     8 GiB gp3
- Subnet:      subnet-0ed69202000fc6f62 (lab-public-a)
- Security gp: sg-0d9c0f37446e20b71 (lab-sg-web)
- Private IP:  10.0.1.65
- Public IP:   44.252.44.159 (auto-assigned, not Elastic)
- Public DNS:  ec2-44-252-44-159.us-west-2.compute.amazonaws.com
- NIC:         ens5 (predictable naming — not eth0)
- IMDSv2:      Required
- IAM role:    none yet (Phase 3)

**Notes**
- Auto-assign public IP defaults to DISABLED on manually created subnets. It
  was explicitly enabled. Without it the instance is unreachable regardless of
  routing and security group configuration, and nothing in the console
  indicates a problem.
- Public DNS name resolves because DNS hostnames was enabled on the VPC. That
  field is blank otherwise.
- No Elastic IP. The public IP changes on every stop/start (not on reboot). An
  EIP is free while attached to a RUNNING instance but billed (~$3.60/mo) when
  unattached or attached to a stopped instance. Accepting the churn is cheaper
  and more realistic. Planned break/fix: stale IP after restart, and SSH host
  key mismatch when an IP is recycled.
- IMDSv2 is required, so bare curl against 169.254.169.254 without a session
  token fails. Older tutorials will not work as written.
- SSH user is `ubuntu`. A wrong username returns "Permission denied
  (publickey)", indistinguishable at first glance from a key problem.
- No swap configured. With 908 MiB RAM and no swap, memory pressure kills
  processes outright rather than degrading. Candidate OOM break/fix scenario.

**Cost from this point forward**

| Item     | Rate          | Per day | Per week |
|----------|---------------|---------|----------|
| t3.micro | ~$0.0104/hr   | ~$0.25  | ~$1.75   |
| 8 GB gp3 | ~$0.08/GB-mo  | ~$0.02  | ~$0.15   |

Stopped instances bill no compute — EBS only, ~$0.02/day.
Operating habit: stop the instance at the end of each session.

### Flask application — 2026-08-27
- Path:   /opt/lab-api
- Venv:   /opt/lab-api/venv
- Server: gunicorn 26.2.0, bind 0.0.0.0:5000
- Stack:  Flask 3.1.3, Werkzeug 3.1.8, Python 3.14

**Endpoints**

| Method | Path      | Response |
|--------|-----------|----------|
| GET    | /         | 200 — service name, hostname |
| GET    | /health   | 200 — status and UTC timestamp |
| POST   | /api/echo | 200 — echoes JSON body; 400 if body is not valid JSON |

**Verification**
- curl from local machine: all four cases pass, including the deliberate 400.
- Postman collection `lab-api` with collection variable {{base_url}}.
  Exported to docs/postman/lab-api.postman_collection.json.
- Response {"host":"ip-10-0-1-65"} confirms traffic traversed
  IGW → lab-rt-public → lab-public-a → sg-0d9c0f37446e20b71:5000 → gunicorn.

**Notes**
- PEP 668: Ubuntu 24.04+ marks system Python as externally managed, so pip
  installs outside a venv fail with externally-managed-environment. The
  --break-system-packages flag does what its name says; a venv is the correct
  answer.
- gunicorn binds 0.0.0.0, not 127.0.0.1. Binding loopback makes the app work
  perfectly on the instance and invisible from outside — a failure that
  presents as a security group problem and is not one.
- Postman export writes an empty string for base_url. Anyone importing the
  collection must set it to http://<instance-ip>:5000 before running.

### systemd service — 2026-08-27
- Unit:  /etc/systemd/system/lab-api.service
- State: active (running), enabled
- Main PID 2115 (gunicorn arbiter), 2 sync workers
- Memory at idle: ~22 MB

**Verified**
- Survives SSH logout: exited the session, /health continued responding.
- Survives reboot: confirmed. `sudo reboot` at 18:46 UTC; /health responded at
  18:47:08 UTC with no manual intervention. Public IP unchanged — reboot does
  not reassign it; only stop/start does.

**Directive reasoning**
- `Type=notify` — gunicorn signals readiness to systemd instead of systemd
  assuming readiness at fork. Confirmed by Status: "Gunicorn arbiter booted".
- `After=` / `Wants=network-online.target` — prevents a boot-time bind failure
  on 0.0.0.0:5000 before the interface is up.
- `User=ubuntu` — runs unprivileged. The systemd default is root; a web app
  does not need it.
- `Environment="PATH=/opt/lab-api/venv/bin"` — systemd starts with a minimal
  environment. Without this the venv interpreter is not found.
- `Restart=on-failure` + `RestartSec=5` — restarts on crash, not on clean
  exit. The delay prevents a crash loop from saturating the CPU.
- `WantedBy=multi-user.target` — the target `enable` symlinks into. This is
  what makes the service start at boot. Confirmed by the symlink created at
  /etc/systemd/system/multi-user.target.wants/lab-api.service.

**Note for Phase 4**
`Restart=on-failure` makes a naive "kill the process" failure uninteresting —
systemd resurrects it in 5 seconds. Realistic service-failure scenarios are
ones systemd cannot fix: bad config, missing dependency, port already bound.
Those produce a visible restart loop in the journal, which is the artifact
worth reading.

---

## Phase 2 complete — 2026-08-27

| Resource   | ID / value                       |
|------------|----------------------------------|
| Key pair   | lab-key                          |
| Instance   | i-0a5d4684f2c4f3008 (lab-web-01) |
| Public IP  | 44.252.44.159 (ephemeral)        |
| Private IP | 10.0.1.65                        |
| Service    | lab-api.service (enabled)        |
| API port   | 5000                             |

Deferred deliberately: 114 pending apt updates (84 security). Left unpatched
so the reboot test measured systemd behavior only. Patching is itself a
planned scenario — a kernel update requires a reboot, which is a realistic
production constraint.

Disk at 34.8% of 6.61 GB after venv and packages. ~4.3 GB free — small enough
that the disk-exhaustion scenario is easy to stage.
