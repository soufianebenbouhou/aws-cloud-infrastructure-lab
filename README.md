# AWS Cloud Infrastructure Lab

A hands-on AWS environment built to practice infrastructure configuration and
systematic troubleshooting. The goal is not to build something that works —
it's to build something, break it deliberately, diagnose it from symptoms, and
document the runbook.

Every resource was created manually through the console rather than with
Terraform or CloudFormation, so that each failure mode is one I configured and
can explain.

**Status:** Phases 1 and 2 complete. Phase 3 (S3 + IAM) and Phase 4
(break/fix scenarios) in progress.

---

## Architecture
Internet
│
▼
Internet Gateway (lab-igw)
│
▼
VPC 10.0.0.0/16 (lab-vpc, us-west-2)
│
├── public subnet 10.0.1.0/24 (us-west-2a)
│ └── EC2 t3.micro — Ubuntu 26.04 LTS
│ └── Flask API behind gunicorn, port 5000, managed by systemd
│
└── private subnet 10.0.2.0/24 (us-west-2b) — reserved, empty

No NAT Gateway, no RDS, no load balancer. Each of those costs real money and
none is required to demonstrate the failure modes this lab is about.

---

## Built so far

**Networking**
- VPC with a manually chosen /16 and DNS hostnames enabled
- Public and private subnets in separate availability zones
- Internet gateway, attached
- Separate route tables for each subnet, both explicitly associated
- Security group: SSH restricted to a single source address, HTTP and the API
  port open

**Compute**
- EC2 instance in the public subnet, key-pair SSH access
- Flask application served by gunicorn in a virtual environment
- Managed as a systemd unit so it survives logout and reboot

**API**

| Method | Path        | Behavior                                            |
|--------|-------------|-----------------------------------------------------|
| GET    | `/`         | 200 — service name and hostname                     |
| GET    | `/health`   | 200 — status and UTC timestamp                      |
| POST   | `/api/echo` | 200 — echoes the JSON body; 400 if the body is not valid JSON |

Verified with curl and with a Postman collection
(`docs/postman/lab-api.postman_collection.json`). Set the `base_url`
collection variable to `http://<instance-ip>:5000` before running it.

---

## In progress

- S3 bucket accessed from the instance through an IAM instance role, with no
  credentials on disk
- CloudWatch log delivery
- Ten documented break/fix incidents: security group, network ACL, routing,
  IAM permissions, DNS, TLS, disk exhaustion, service failure, and others

Each incident is documented in the same structure:
**Symptoms → Investigation → Commands → Evidence → Root cause → Fix → Prevention**

---

## Design decisions

A few choices that are worth stating explicitly, since they shaped everything
downstream:

**"VPC only" instead of the "VPC and more" wizard.** The wizard provisions a
NAT Gateway by default at roughly $32/month. It also hides the wiring between
subnets, route tables, and the internet gateway — which is exactly the wiring
the break/fix scenarios depend on understanding.

**Cost guardrails before any resource existed.** A zero-spend budget was
created first. Reading its exported JSON showed no `CostTypes` block, meaning
the API defaults apply and `IncludeCredit` is true — so the budget measures
*net* cost and stays silent while promotional credits absorb charges. That's
the correct behavior for catching real charges, but it means credit burn needs
a separate gross-spend tripwire. The console does not surface this; only the
JSON does.

**Explicit route table associations for both subnets.** A subnet with no
explicit association silently inherits the VPC's main route table, which has
no route to the internet gateway. Such a subnet appears correctly configured
in every list view and is unreachable. Making both associations explicit
removes that ambiguity — and the inherited-main-table failure is on the
break/fix list precisely because it is invisible.

**No Elastic IP.** An EIP is free while attached to a running instance and
billed when it isn't. Accepting a changing public IP is cheaper and produces
two realistic failures: stale addresses after a restart, and SSH host key
mismatches when an address is recycled.

---

## Repository
docs/
build-log.md chronological record of every resource,
decision, cost, and incident
postman/
lab-api.postman_collection.json API verification collection

Account identifiers and the SSH source address are deliberately excluded from
this repository.

---

## Stack

AWS (VPC · EC2 · S3 · IAM · CloudWatch · Budgets) · Ubuntu Linux · Python ·
Flask · gunicorn · systemd · Postman
