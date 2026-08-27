# Incident 05 — IAM role detached at runtime; app keeps working, then won't recover

**Date:** 2026-08-27
**Layer:** IAM / SDK credential resolution
**Severity:** Partial outage on S3-backed endpoints. Health checks stayed green
throughout. Recovery required a service restart that the console gave no
indication was needed.

---

## Symptoms

Immediately after the IAM role was detached from the running instance:

- `/health`, `/`, and `/api/echo` return 200 — unaffected
- `/api/records` returns **200** and `/api/store` returns **201** — still working
- Response times 53ms and 66ms, indicating real successful S3 calls
- Meanwhile, on the same instance in the same minute:
  `aws sts get-caller-identity` → `NoCredentials: Unable to locate credentials`

After a service restart:

- `/api/records` and `/api/store` return **HTTP 500** with a generic HTML error
  page, not the structured JSON 502 the application's error handler produces
- Journal shows `botocore.exceptions.NoCredentialsError: Unable to locate credentials`

After the role was reattached:

- CLI recovered immediately
- **The application continued returning 500 for over two minutes**
- Only a second restart restored it

---

## Investigation

**1. The application worked while the CLI failed.**
23:31:45 — application

GET /api/records → HTTP 200, 4 keys, real 0m0.053s

23:32 — CLI, same instance

aws sts get-caller-identity
→ NoCredentials: Unable to locate credentials
Same host, same second, opposite results. This ruled out a network or IMDS
outage and pointed at process state.

**2. IMDS confirmed the role was gone.**
curl -H "X-aws-ec2-metadata-token: $TOKEN"
http://169.254.169.254/latest/meta-data/iam/security-credentials/
→ HTTP 404

**3. IMDS itself was healthy.**
curl -H "X-aws-ec2-metadata-token: $TOKEN"
http://169.254.169.254/latest/meta-data/instance-id
→ i-0a5d4684f2c4f3008
Only the credentials path was gone. The metadata service was fine.

**4. Restarting the service reproduced the failure.**
The restart cleared whatever the running process was holding. Post-restart the
app failed exactly as the CLI did — confirming cached credentials as the cause
of the apparent immunity.

**5. The failure was a 500, not the application's 502.**
The handler catches `ClientError`. `NoCredentialsError` is a different
exception class and propagated uncaught.

---

## Commands

```bash
aws sts get-caller-identity
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -w "\nHTTP %{http_code}\n" -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl -s -w "\nHTTP %{http_code}\n" -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
sudo systemctl restart lab-api
sudo journalctl -u lab-api --since "2 minutes ago" --no-pager | tail -20
```

---

## Evidence

Timeline:

| Time (UTC) | State | Application | CLI |
|---|---|---|---|
| 23:31:45 | role detached | 200 / 201, 53-66ms | — |
| 23:32 | role detached | — | NoCredentials |
| 23:33:39 | after restart | HTTP 500 | NoCredentials |
| ~23:37 | role reattached | HTTP 500 | works |
| 23:39:35 | role reattached | HTTP 500 | works |
| ~23:41 | after restart | **HTTP 200** | works |

Stack trace after restart:
File "/opt/lab-api/venv/lib/python3.14/site-packages/botocore/signers.py", line 200, in sign
auth.add_auth(request)
File "/opt/lab-api/venv/lib/python3.14/site-packages/botocore/auth.py", line 429, in add_auth
raise NoCredentialsError()
botocore.exceptions.NoCredentialsError: Unable to locate credentials

---

## Root cause

Detaching an IAM role stops IMDS from serving credentials. It does not affect
credentials a running process already holds.

boto3 fetches temporary credentials from IMDS and caches them in memory with
their expiration. The Flask application, running continuously under gunicorn,
kept using its cached set and continued serving S3 requests normally. The AWS
CLI is a fresh process on every invocation, has no cache, and failed
immediately.

Restoring the role was equally ineffective on the running process. The
application constructs its S3 client at module import
(`s3 = boto3.client("s3", ...)`), once per worker. A worker that started
without credentials holds a client permanently in that state; reattaching the
role does not retroactively repair an already-constructed client.

**Both directions required a restart to take effect. Neither is visible in the
console, which showed the role as attached and correct throughout the second
half of the outage.**

The 500 rather than a 502 has its own cause: the application catches
`ClientError`, which covers errors returned *by* AWS. `NoCredentialsError` is
raised during request signing, before any HTTP request is made — the request
never reaches AWS, so there is no AWS error code. It propagated uncaught and
Flask returned a generic 500 with an HTML body containing no diagnostic
information.

---

## Fix

EC2 → Instances → select instance → Actions → Security → Modify IAM role →
select the role → Update IAM role.

Then **restart the application**:
```bash
sudo systemctl restart lab-api
```
The restart is required. Reattaching alone left the service failing.

---

## Prevention

- **Detaching an IAM role is not an immediate revocation.** A running process
  keeps working until its cached credentials expire — potentially hours. In a
  security incident where a role must be revoked urgently, detaching is not
  sufficient. Attaching an explicit `Deny` policy takes effect on the next API
  call, because it is evaluated at the service, not at the client.
- **Reattaching a role does not restore a broken application.** Clients built
  before credentials were available stay broken. Any IAM change affecting a
  long-running process should be followed by a restart, and monitoring should
  not assume recovery once the console looks correct.
- **The console is not a source of truth for the running process.** For the
  final four minutes of this outage, EC2 showed the role attached and the CLI
  confirmed credentials worked, while the application returned 500 on every
  request.
- **Health checks did not detect this.** `/health` does not touch AWS, so it
  returned 200 throughout. Any monitor watching only that endpoint would have
  reported the service green during a partial outage. A health endpoint that
  exercises a lightweight dependency call would have caught it — at the cost of
  making the health check itself fail on transient AWS issues, which is a
  deliberate tradeoff.
- **Catch `NoCredentialsError` separately from `ClientError`.** They are
  different failure classes: authentication versus authorization. Returning a
  structured error for both makes the difference visible from the client
  instead of collapsing one into an opaque HTML 500. Improvement to make:

```python
  from botocore.exceptions import ClientError, NoCredentialsError

  except NoCredentialsError:
      app.logger.error("No AWS credentials available")
      return jsonify({"error": "storage unavailable",
                      "reason": "no credentials"}), 503
```
  A 503 is more accurate than a 502 here: the dependency is unreachable
  because the caller cannot authenticate, not because the upstream failed.
- **`sts get-caller-identity` is the fastest credential check.** It requires no
  permissions beyond being authenticated, so it isolates authentication from
  authorization in one command. Compare its result against the application's
  behavior — if they disagree, the difference is process state, not
  configuration.
- **Check the IMDS credentials path directly when diagnosing.** A 404 on
  `/latest/meta-data/iam/security-credentials/` while `/latest/meta-data/instance-id`
  returns normally proves the role is detached rather than the metadata service
  being unreachable.
  
