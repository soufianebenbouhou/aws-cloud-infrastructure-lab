# Incident 08 — Root filesystem at 100%; the application never noticed

**Date:** 2026-08-28
**Layer:** Host / filesystem
**Severity:** No user-facing outage. Significant operational impact:
package management crashed and left corrupted state.

---

## Symptoms

Root filesystem filled to 0 bytes available:
/dev/root 6.7G 6.6G 0 100% /

**What kept working:**
GET /health → 200
GET /api/records → 200
POST /api/store → 201 (wrote a new object to S3)
systemctl status lab-api → active (running)
systemctl status amazon-cloudwatch-agent → active (running)
touch /tmp/testfile → succeeded

**What failed:**
echo "test" > /home/ubuntu/testwrite
-bash: echo: write error: No space left on device

sudo apt update
Err:2 ... InRelease
Error writing to file - write (28: No space left on device)
Segmentation fault (core dumped)
Reading package lists... Error!
Error: Write error - write (28: No space left on device)
Error: IO Error saving source cache
Error: The package lists or status file could not be parsed or opened.

sudo fallocate -l 200M /var/tmp/ballast4
fallocate: fallocate failed: No space left on device

---

## Investigation

**1. The application was unaffected, and that is the finding.**
A disk-full condition is often assumed to take an application down. This one
did not, because the service holds an already-open listening socket, writes no
files during request handling, and logs to journald rather than to its own log
file. Nothing in its request path requires disk allocation.

**2. `apt` did not fail gracefully — it segfaulted.**
It hit ENOSPC mid-write and crashed, then reported that the package lists or
status file could not be parsed. Freeing the disk does not automatically repair
that state; `apt` may need its cache cleared and rebuilt afterward.

**3. `/tmp` writes still succeeded.**
tmpfs 455M 0 455M 0% /tmp
`/tmp` is tmpfs — backed by RAM, not by the root filesystem. `touch /tmp/file`
succeeds while `/home` writes fail. A "can I write a file?" check performed in
`/tmp` reports a healthy system during a full-disk outage.

**4. Inode exhaustion ruled out.**
df -i /
Filesystem Inodes IUsed IFree IUse%
/dev/root 878080 111910 766170 13%
A filesystem can return "No space left on device" with free bytes remaining if
inodes are exhausted — typically millions of small files. Different cause,
different fix. Checking `df -i` alongside `df -h` takes one command and
eliminates it.

**5. The CloudWatch agent log contained a red herring.**
2026-08-27T22:18:12Z E! [outputs.cloudwatchlogs] ... i/o timeout
2026-08-27T22:20:57Z E! cloudwatch: ... i/o timeout
All of those entries are timestamped 22:18–22:21 — during the NACL outage of
Incident 02, roughly two hours earlier. Nothing was logged during the disk-full
window. Reading the tail of a log file without checking timestamps would have
led to investigating a network problem that had already been resolved.

**6. Located the consumption top-down.**
sudo du -h --max-depth=1 / | sort -h | tail
4.3G /var ← largest
2.1G /usr
625M /snap

sudo du -h --max-depth=1 /var | sort -h | tail
3.8G /var/tmp ← the culprit
369M /var/lib
149M /var/cache
Two commands narrowed 6.7 GB to a single directory.

---

## Commands

```bash
df -h /
df -h                 # all filesystems — reveals tmpfs mounts
df -i /               # inode usage
sudo du -h --max-depth=1 / 2>/dev/null | sort -h | tail -10
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h | tail -10
ls -lh /var/tmp/
sudo journalctl --disk-usage
```

---

## Evidence

Fill progression and application response at each stage:

| Available | Use% | /health |
|---|---|---|
| 3.8 G | 44% | 200 |
| 1.8 G | 74% | 200 |
| 743 M | 90% | 200 |
| 43 M | 100% | 200 |
| 0 | 100% | **200** |

Journal size remained small (20.5 M) and was not a factor.

---

## Root cause

Deliberate: 3.8 GB of ballast files created in `/var/tmp` with `fallocate`.

In production the equivalent causes are unrotated application logs, a runaway
debug log, accumulated core dumps, an unbounded `/var/cache`, or a journal
without a size limit.

---

## Fix

```bash
sudo rm /var/tmp/ballast*
df -h /
```

Recovery was immediate and required no service restart. The application had
never entered a bad state, so there was nothing to restore.

Note: `apt` may still be broken after the disk is freed, because its cache was
corrupted mid-write. If `apt update` continues to fail:
```bash
sudo rm -rf /var/lib/apt/lists/*
sudo apt update
```
In this case apt recovered without intervention: `apt update` succeeded on the
next run once space was available. The corruption was to a partially-written
temporary file rather than to the persistent package database. Worth verifying
rather than assuming either way.
---

## Prevention

- **A full disk does not necessarily produce a user-facing outage.** This
  service continued returning 200 at 0 bytes free. The damage was to
  operational capability — package management, log writes, config changes —
  not to request handling. Both matter, but only one is visible to monitoring
  that watches HTTP status codes.
- **Alert on disk usage, not on symptoms.** By the time something visibly
  fails, the filesystem has been full for a while. The CloudWatch agent is
  already collecting `disk used_percent` on `/` at 60s intervals; an alarm at
  80% would have fired long before 100%.
- **`/tmp` is tmpfs and does not reflect root filesystem state.** Any health
  check that writes to `/tmp` to verify disk health will pass during a full
  disk. Write the probe file to the filesystem you actually care about.
- **Check `df -i` as well as `df -h`.** Inode exhaustion produces the identical
  "No space left on device" error with free bytes available, and needs a
  different fix.
- **Check timestamps before treating log entries as evidence.** The CloudWatch
  agent's error tail was two hours stale and belonged to a different, already
  resolved incident. `tail` shows the last lines, not the recent lines.
- **`apt` can crash and leave corrupted state.** It segfaulted rather than
  exiting cleanly. Freeing space does not automatically repair the package
  cache.
- **`du --max-depth=1 | sort -h` narrows fast.** Two invocations —
  root, then the largest subdirectory — located 3.8 GB in one directory
  without any guessing.
- **Set `SystemMaxUse` in `/etc/systemd/journald.conf`.** The journal was only
  20.5 MB here, but an unbounded journal on a long-running host is a common
  cause of exactly this incident.
