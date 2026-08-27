# Incident 04 — Writes succeed, listing fails, same bucket and same role

**Date:** 2026-08-27
**Layer:** IAM / S3
**Severity:** Partial. One API endpoint returns 502; the rest of the service
works normally.

---

## Symptoms

- `GET /api/records` returns 502 with `{"aws_error":"AccessDenied","error":"list failed"}`
- `POST /api/store` returns 201 and successfully writes to the same bucket
- No change to the instance, the role attachment, or the application
- `sts:GetCallerIdentity` still returns the expected assumed-role ARN
- GET /api/records
< HTTP/1.1 502 BAD GATEWAY
{"aws_error":"AccessDenied","error":"list failed"}

POST /api/store {"event":"during-incident-04"}
{"bucket":"lab-api-storage-929214127133",
"key":"records/2026-08-27/4879c9d8-...json","stored":true}

Reads and writes work. Listing does not. This reads as intermittent
authorization failure, which is what makes it confusing.

---

## Investigation

**1. Credentials were valid.**
aws sts get-caller-identity
→ arn:aws:sts::929214127133:assumed-role/lab-ec2-s3-role/i-0a5d4684f2c4f3008
The role was attached and STS was issuing credentials. Not an authentication
problem.

**2. Reproduced on the CLI to isolate from the application.**
The 502 came from the app's error handler. Running the same operations
directly on the instance removed the application as a variable.

**3. Tested each operation type separately.**

| Operation | IAM action | Result |
|---|---|---|
| `aws s3 ls s3://bucket` | s3:ListBucket | **AccessDenied** |
| `aws s3 cp file s3://bucket/key` | s3:PutObject | success |
| `aws s3 cp s3://bucket/key file` | s3:GetObject | success |

Object operations passed; the bucket operation failed. That split pointed at
resource scope rather than at the action list.

**4. Read the error text carefully.**
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:sts::929214127133:assumed-role/lab-ec2-s3-role/i-0a5d4684f2c4f3008
is not authorized to perform: s3:ListBucket
on resource: "arn:aws:s3:::lab-api-storage-929214127133"
because no identity-based policy allows the s3:ListBucket action

The message names the resource ARN the request was evaluated against:
`arn:aws:s3:::lab-api-storage-929214127133`, with no trailing `/*`. Comparing
that string to the policy's Resource field identified the fault directly.

---

## Commands

```bash
aws sts get-caller-identity
aws s3 ls s3://lab-api-storage-929214127133 --region us-west-2
echo '{"x":1}' > /tmp/t.json
aws s3 cp /tmp/t.json s3://lab-api-storage-929214127133/t.json --region us-west-2
aws s3 cp s3://lab-api-storage-929214127133/t.json /tmp/back.json --region us-west-2
```

---

## Evidence

Policy at time of failure — one statement, one ARN:
```json
{
  "Action": ["s3:ListBucket","s3:GetObject","s3:PutObject","s3:DeleteObject"],
  "Resource": "arn:aws:s3:::lab-api-storage-929214127133/*"
}
```

---

## Root cause

`s3:ListBucket` operates on the **bucket**; `GetObject`, `PutObject`, and
`DeleteObject` operate on **objects**. They require different resource ARNs.

| Action | Resource type | Required ARN |
|---|---|---|
| s3:ListBucket | bucket | `arn:aws:s3:::bucket-name` |
| s3:GetObject | object | `arn:aws:s3:::bucket-name/*` |
| s3:PutObject | object | `arn:aws:s3:::bucket-name/*` |
| s3:DeleteObject | object | `arn:aws:s3:::bucket-name/*` |

The policy had been collapsed into a single statement using only the
object-level ARN. `arn:aws:s3:::bucket-name/*` matches objects inside the
bucket; it does not match the bucket itself. `ListBucket` therefore found no
matching Allow and was denied, while every object operation matched and
succeeded.

The inverse mistake produces the mirror symptom: with only the bare bucket ARN,
listing works and every read and write fails.

---

## Fix

Split the policy into two statements, each with the ARN form its actions
require:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListLabBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::lab-api-storage-929214127133"
    },
    {
      "Sid": "ReadWriteLabObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject"],
      "Resource": "arn:aws:s3:::lab-api-storage-929214127133/*"
    }
  ]
}
```

Recovery was immediate on save. No role reattachment, no application restart —
IAM policy changes apply to the next API call.

---

## Prevention

- **The error message contains the answer.** It names the principal, the
  action, and the exact resource ARN evaluated. Comparing that ARN string
  against the policy's Resource field is faster than reasoning about the
  policy from scratch. Read the whole message before opening the console.
- **The API name and the IAM action name differ.** The error reports the
  `ListObjectsV2` operation but the denied action `s3:ListBucket`. Searching
  for "ListObjectsV2 permission" finds nothing useful. The IAM action name is
  the one that matters, and the error states it explicitly.
- **Partial failure points at resource scope, not at the action list.** If the
  role were missing entirely, everything would fail. If the action were
  missing, that one operation would fail everywhere. One operation failing
  while its siblings succeed against the same bucket indicates a resource ARN
  mismatch.
- **Bucket-level and object-level actions need separate statements.** Any S3
  policy granting both listing and object access requires two ARNs. Collapsing
  them into one statement breaks one half silently. Other bucket-level actions
  with the same requirement include `s3:GetBucketLocation`,
  `s3:ListBucketMultipartUploads`, and `s3:GetBucketVersioning`.
- **Test operations independently after any policy change.** The application
  exercised `PutObject` on one endpoint and `ListBucket` on another, so half
  the service kept working and the failure looked intermittent. A post-change
  check that only writes would have reported success.
- **The IAM Policy Simulator** (IAM → Policy simulator) evaluates an action
  against a resource ARN without making a live call, and would have shown this
  denial before deployment.
  
