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
| 02 | | |
| 03 | | |
| 04 | | |
| 05 | | |
| 06 | | |
| 07 | | |
| 08 | | |
| 09 | | |
| 10 | | |
