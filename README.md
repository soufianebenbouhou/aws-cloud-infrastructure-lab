# AWS Cloud Infrastructure Lab

I built a small AWS environment by hand and then broke it ten times on purpose
to practice diagnosing failures.

Everything was created through the console rather than Terraform, so every
failure mode here is one I configured myself and can explain. Each incident was
introduced against a working baseline, diagnosed from symptoms, then reverted
before the next one.

**Status:** finished. Ten incidents documented.

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
│ └── Flask API behind gunicorn, port 5000, under systemd
│ └── boto3 → IMDS → temporary credentials → S3
│
└── private subnet 10.0.2.0/24 (us-west-2b) — reserved, empty

No NAT Gateway, no RDS, no load balancer. They all cost money and none of them
were needed for what I was trying to practice.

For observability: CloudWatch agent shipping syslog and auth.log on 7-day
retention, plus memory and disk metrics, which EC2 doesn't report on its own.
VPC Flow Logs on the public subnet.

---

## The incidents

Each one is written up the same way: Symptoms, Investigation, Commands,
Evidence, Root cause, Fix, Prevention.

| # | Incident | Layer |
|---|----------|-------|
| 01 | [API unreachable, SSH unaffected](docs/runbooks/01-security-group-source-mismatch.md) | Security group |
| 02 | [Request accepted, reply dropped](docs/runbooks/02-nacl-stateless-return-traffic.md) | Network ACL |
| 03 | [New instance unreachable, identical config](docs/runbooks/03-subnet-inherits-main-route-table.md) | Routing |
| 04 | [Writes succeed, listing fails](docs/runbooks/04-iam-resource-arn-mismatch.md) | IAM / S3 |
| 05 | [Role detached; app keeps working, then won't recover](docs/runbooks/05-iam-role-detached-credential-caching.md) | IAM / SDK |
| 06 | [Healthy locally, refused from the network](docs/runbooks/06-service-bound-to-loopback.md) | Application / bind address |
| 07 | [Service in a restart loop](docs/runbooks/07-service-restart-loop-import-error.md) | Application / systemd |
| 08 | [Filesystem at 100%, application unaffected](docs/runbooks/08-disk-exhaustion.md) | Host / filesystem |
| 09 | [Process killed by the kernel, no application log](docs/runbooks/09-oom-kill.md) | Host / kernel |
| 10 | [SSH refuses to connect, host identification changed](docs/runbooks/10-ssh-host-key-mismatch.md) | SSH / trust |

---

## What surprised me

Three of these produce the exact same symptom. Incidents 01, 02 and 03 all look
like a connection that hangs and times out. One is a security group dropping
packets, one is a NACL blocking the reply, one is a subnet with no route to the
internet. From the client there's no way to tell them apart, so all three came
down to evidence the client can't see.

Timeout and "Connection refused" turned out to be the most useful thing on the
screen. A timeout means something ate the packet, so it's a security group,
NACL, or route. A refusal means the packet got there and nothing was listening,
so it's the process. Incident 06 took one command to solve because the error
said refused instead of hanging.

Incident 05 was the one I didn't expect. I detached the IAM role and the app
kept working for minutes on credentials boto3 had already cached in memory,
while the CLI on the same box failed instantly. Then reattaching the role
didn't fix it, because the client object had been built without credentials and
stayed that way until I restarted the service. The whole time the console
showed the role attached and correct.

Two of the failures wrote nothing to the application log at all. The OOM kill
left "-- No entries --" in the journal, because SIGKILL can't be caught and the
process never gets a chance to log anything. The security group change left
nothing because the requests never arrived. One needed dmesg, the other needed
flow logs.

Filling the disk to zero bytes didn't take the API down. It kept returning 200
and 201. What actually broke was apt, which segfaulted partway through a write.
Anything monitoring HTTP status codes would have called the box healthy.

One more thing worth mentioning: while I was working on the disk incident, the
CloudWatch agent log was full of connection errors. They were two hours stale
and belonged to the NACL incident I'd already fixed. tail gives you the last
lines, not the recent ones, and I nearly went chasing a network problem that
didn't exist.

---

## Some decisions I made and why

I used "VPC only" instead of the "VPC and more" wizard. The wizard adds a NAT
Gateway by default, which is around $32/month, and it hides the wiring between
subnets, route tables and the gateway. That wiring is the whole point of three
of these incidents.

I set up the budget before creating anything. When I downloaded the budget
template as JSON there was no CostTypes block in it, which means the API
defaults apply and IncludeCredit is true. So the budget measures net cost and
will stay quiet the entire time promotional credits are covering the bill. That
turned out to be the right behavior for catching real charges, but it means
credit burn needs a second budget with credits excluded. The console doesn't
show any of this anywhere.

There are no credentials on the host. The instance gets to S3 through an IAM
role and the metadata service. No ~/.aws/credentials, no access key, nothing in
the code. The policy allows three actions on one bucket, so if the app were
compromised that's the whole blast radius.

I skipped the Elastic IP. It's free while attached to a running instance and
billed when it isn't, and living with a changing public IP is both cheaper and
more realistic. It also generated two of the incidents here.

Both subnets are explicitly associated with a route table. If you skip that, the
subnet quietly inherits the VPC main route table, which has no internet route.
It looks completely normal in every list view and nothing can reach it. That's
Incident 03.

---

## Repo layout
docs/
build-log.md everything I built, in order, with costs
runbooks/
README.md index
01-…10-…md the incidents
postman/
lab-api.postman_collection.json API tests

The API has five endpoints: two for health and info, one echo endpoint with a
deliberate 400 path, and two that read and write S3. Tested with curl and
Postman. If you import the collection, set the `base_url` variable to
`http://<instance-ip>:5000` first.

Account IDs and my home IP are kept out of this repo.

---

## Stack

AWS (VPC, EC2, S3, IAM, CloudWatch, VPC Flow Logs, Budgets), Ubuntu, Python,
Flask, gunicorn, systemd, boto3, Postman
