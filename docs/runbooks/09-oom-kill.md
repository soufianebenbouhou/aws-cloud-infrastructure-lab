# Incident 09 — Process killed by the kernel; no application log records it

**Date:** 2026-08-28
**Layer:** Host / kernel memory management
**Severity:** No service impact in this case. The mechanism can terminate any
process on the host, including the application.

---

## Symptoms

- A process terminated abruptly with no traceback and no exit message beyond
  the shell's `Killed`
- `lab-api` unaffected: `active (running)` throughout, uninterrupted since
  00:38:51
- `/health` returned 200 during and after
- **The application journal contained no entries at all for the period:**
- sudo journalctl -u lab-api --since "5 minutes ago" --no-pager | tail -5
-- No entries --
  - Memory returned to normal by itself

---

## Investigation

**1. The absence of application logs is the diagnostic signal.**
A process died and nothing in the application journal recorded it. That is
expected: the kernel sends SIGKILL, which cannot be caught, blocked, or
handled. The process gets no opportunity to log, flush, or clean up. Any
investigation that stays inside application logs finds nothing.

**2. The record is in the kernel ring buffer.**
```bash
sudo dmesg -T | grep -i "out of memory\|killed process"
sudo journalctl -k --since "10 minutes ago" | grep -i "oom"
```

**3. Pre-incident OOM scores predicted the victim.**
gunicorn worker 4694: 710
gunicorn worker 4699: 710
gunicorn master 4693: 684
sshd: 0
`/proc/<pid>/oom_score` is readable at any time. The allocating process grew
past every gunicorn worker and became the highest scorer, so it was selected.
`sshd` at 0 was never a candidate — systemd applies a protective
`OOMScoreAdjust` to it, which is why the SSH session survived.

**4. Memory recovered without intervention.**
before: 908Mi total, 481Mi used, 193Mi free
after: 908Mi total, 449Mi used, 440Mi free

---

## Commands

```bash
free -h
ps -eo pid,comm,rss --sort=-rss | head -8
for pid in $(pgrep gunicorn); do echo -n "$pid: "; cat /proc/$pid/oom_score; done
sudo dmesg -T | grep -i "out of memory\|killed process" | tail -10
sudo dmesg -T | grep -A 60 "\[  pid  \]" | head -70   # full candidate table
sudo journalctl -k --since "10 minutes ago" --no-pager | grep -i "oom"
```

---

## Evidence
[Fri Aug 28 00:50:17 2026] python3 invoked oom-killer:
gfp_mask=0x140dca(GFP_HIGHUSER_MOVABLE|__GFP_ZERO|__GFP_COMP),
order=0, oom_score_adj=0

[Fri Aug 28 00:50:17 2026] oom-kill:
constraint=CONSTRAINT_NONE, nodemask=(null), cpuset=/, mems_allowed=0,
global_oom,
task_memcg=/user.slice/user-1000.slice/session-11.scope,
task=python3, pid=5460, uid=1000

[Fri Aug 28 00:50:17 2026] Out of memory: Killed process 5460 (python3)
total-vm:555944kB anon-rss:513116kB file-rss:2004kB shmem-rss:0kB
UID:1000 pgtables:1072kB oom_score_adj:0

Reading the fields:

| Field | Value | Meaning |
|---|---|---|
| `anon-rss` | 513116 kB | resident anonymous memory at death — 513 MB |
| `total-vm` | 555944 kB | total virtual address space |
| `constraint` | CONSTRAINT_NONE | system-wide exhaustion, not a cgroup limit |
| `global_oom` | — | every process on the host was a candidate |
| `task_memcg` | user-1000.slice/session-11.scope | victim's cgroup — the SSH session, not system.slice |
| `oom_score_adj` | 0 | no manual bias applied to the victim |

---

## Root cause

Deliberate: a Python loop allocating 50 MB per iteration on a host with
908 MiB RAM and **no swap configured**.

Without swap there is no pressure-relief mechanism. Memory exhaustion goes
directly from "allocation succeeds" to "kernel kills something" with no
intervening slowdown. On a host with swap, the same workload would first cause
severe paging and degraded latency — a visible warning phase. Here there was no
warning: allocations succeeded normally until one didn't.

The kernel selected the victim by `oom_score`, which is driven mainly by
resident memory. The allocating process was by then the largest consumer, so it
was chosen over the gunicorn workers.

---

## Fix

None required — the process was already dead and memory reclaimed
automatically. Verified:

```bash
free -h                    # 440Mi free
systemctl status lab-api   # active (running), uninterrupted
curl localhost:5000/health # 200
```

Had `lab-api` been the victim, `Restart=on-failure` would have restarted it
within 5 seconds, and `systemctl status` would show
`Result: oom-kill` or a SIGKILL result rather than an exit code.

---

## Prevention

- **Never look for an OOM kill in application logs.** SIGKILL cannot be caught,
  so the process logs nothing. `dmesg -T` and `journalctl -k` are the only
  sources. A service that "just disappears" with no error is the signature.
- **`/proc/<pid>/oom_score` is readable before anything goes wrong.** Checking
  it on a memory-constrained host tells you in advance which process the kernel
  would choose. Higher score, higher risk.
- **No swap means no warning phase.** With swap, memory pressure first appears
  as heavy paging and slow response times — an observable degradation before
  anything dies. Without it the failure is instantaneous. EC2 instances have no
  swap by default. Adding a small swap file trades a fast kill for a slow
  degradation, which is usually easier to detect and act on.
- **EC2 does not report memory usage to CloudWatch by default.** The hypervisor
  cannot see inside the guest. The CloudWatch agent installed in Phase 3
  collects `mem_used_percent` at 60s intervals; without it there is no memory
  metric at all and no possible alarm. An alarm at 85% would have given warning
  here.
- **Protect critical processes with `OOMScoreAdjust`.** A negative value in a
  systemd unit lowers a service's kill priority. This is why `sshd` scored 0
  and survived — losing SSH during a memory incident means losing the ability
  to fix it.
- **Set `MemoryMax=` on the service unit to change the failure mode.** A cgroup
  limit means the runaway process is killed inside its own cgroup before the
  host runs out, leaving everything else untouched. The kernel log would then
  show a memcg-constrained kill rather than `constraint=CONSTRAINT_NONE`.
- **The process that triggers the OOM killer is not always the victim.** Here
  they happened to be the same. The kernel selects by score, so a small
  allocation by one process can cause a different, larger process to be killed.
