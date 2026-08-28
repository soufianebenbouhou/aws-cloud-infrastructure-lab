# Runbooks

Ten deliberate failures introduced into a working AWS environment, diagnosed
from symptoms, and documented.

Each incident follows the same structure:

**Symptoms** — what was observed, before knowing the cause
**Investigation** — what was checked, in order, including dead ends
**Commands** — the actual commands run
**Evidence** — real output
**Root cause** — what was actually wrong
**Fix** — what resolved it
**Prevention** — what would stop it recurring, or detect it sooner

The environment was working before each change. Failures were introduced one
at a time against a known-good baseline.

| # | Incident | Layer |
|---|----------|-------|
| 00 | [Cannot register second MFA device](../build-log.md) | IAM |
| 01 | [API unreachable, SSH unaffected](01-security-group-source-mismatch.md) | Security group |
| 02 | [Request accepted, reply dropped](02-nacl-stateless-return-traffic.md) | Network ACL |
| 03 | [New instance unreachable, identical config](03-subnet-inherits-main-route-table.md) | Routing |
| 04 | [Writes succeed, listing fails](04-iam-resource-arn-mismatch.md) | IAM / S3 |
| 05 | [Role detached; app keeps working, then won't recover](05-iam-role-detached-credential-caching.md) | IAM / SDK |
| 06 | [Healthy locally, refused from the network](06-service-bound-to-loopback.md) | Application / bind address |
| 07 | [Service in a restart loop](07-service-restart-loop-import-error.md) | Application / systemd |
| 08 | [Filesystem at 100%, application unaffected](08-disk-exhaustion.md) | Host / filesystem |
| 09 | [Process killed by the kernel, no application log](09-oom-kill.md) | Host / kernel |
| 10 | [SSH refuses to connect, host identification changed](10-ssh-host-key-mismatch.md) | SSH / trust |
