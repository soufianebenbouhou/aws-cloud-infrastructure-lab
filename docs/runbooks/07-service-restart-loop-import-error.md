# Incident 07 — Service in a restart loop; systemd cannot recover it

**Date:** 2026-08-28
**Layer:** Application / systemd
**Severity:** Full outage. `Restart=on-failure` cannot fix a failure that
occurs during startup.

---

## Symptoms

- Nothing listening on port 5000. `curl` returns HTTP 000 (no connection).
- `systemctl status` shows `activating (auto-restart)`, not `active` or `failed`
- The unit cycles through start → crash → wait → start every ~6 seconds
- Confusingly, `Status:` still reads `"Gunicorn arbiter booted"`
 ● lab-api.service - lab-api Flask service
Active: activating (auto-restart) (Result: exit-code) since 00:38:12 UTC; 2s ago
Process: 4498 ExecStart=... (code=exited, status=1/FAILURE)
Main PID: 4498 (code=exited, status=1/FAILURE)
Status: "Gunicorn arbiter booted"

---

## Investigation

**1. `activating (auto-restart)` is its own state.**
Not `active`, not `failed`. The unit is mid-retry. Running `systemctl status`
repeatedly shows a changing PID and elapsed time — the sign of a loop rather
than a stable failure.

**2. The Status line was misleading.**
`Status: "Gunicorn arbiter booted"` is a stale `Type=notify` message. The
gunicorn master signalled readiness, then its workers failed to import the
application module and the master shut down. systemd continued displaying the
last notification it received. A status line reporting success on a service
that is not running is a real trap when reading only the summary.

**3. Two exit codes appear for the same event.**
Process: 4498 ExecStart=... (code=exited, status=1/FAILURE)
lab-api.service: Main process exited, code=exited, status=3/NOTIMPLEMENTED
`status=3` is gunicorn's own code for "worker failed to boot."
`status=1` is the generic unit failure. Neither is a systemd defect.

**4. The journal contained the real cause, buried in stack frames.**
The useful line was the last one, not the first:
ImportError: cannot import name 'nonexistent_thing' from 'flask'
(/opt/lab-api/venv/lib/python3.14/site-packages/flask/init.py)
Reached fastest with:
```bash
sudo journalctl -u lab-api -n 50 --no-pager | grep -i "error\|ImportError"
```

**5. Gunicorn's own error was clear once found.**
[ERROR] Exception in worker process
[ERROR] Worker (pid:4569) exited with code 3.
[ERROR] Shutting down: Master
[ERROR] Reason: Worker failed to boot.
"Worker failed to boot" means the application module could not be imported —
distinct from a runtime error inside a request.

**6. Running the app outside systemd isolated the fault.**
```bash
cd /opt/lab-api && source venv/bin/activate && python app.py

Traceback (most recent call last):
  File "/opt/lab-api/app.py", line 1, in <module>
    from flask import Flask, jsonify, request, nonexistent_thing
ImportError: cannot import name 'nonexistent_thing' from 'flask'
```
Three lines instead of forty. No gunicorn or systemd frames. This step
separates "the code is broken" from "the service manager is misconfigured" and
should come early, not late.

---

## Commands

```bash
sudo systemctl status lab-api --no-pager
sudo journalctl -u lab-api --since "5 minutes ago" --no-pager | tail -40
sudo journalctl -u lab-api -n 50 --no-pager | grep -i "error\|ImportError"
cd /opt/lab-api && source venv/bin/activate && python app.py
curl -s -w "\nHTTP %{http_code}\n" --max-time 5 http://localhost:5000/health
```

---

## Evidence

Restart cycle, ~6s apart, matching `RestartSec=5`:
00:38:12 ImportError → Worker failed to boot → Main process exited
00:38:18 ImportError → same
00:38:24 ImportError → same
00:38:29 ImportError → same
00:38:51 fixed, active (running)

Failure path through the stack:
gunicorn/util.py:420 import_app
importlib/init.py:88 import_module
/opt/lab-api/app.py:1 from flask import ..., nonexistent_thing
→ ImportError

---

## Root cause

A non-existent name was added to the import statement on line 1 of `app.py`.
The module could not be imported, so every gunicorn worker failed to boot and
the master exited. systemd restarted it under `Restart=on-failure`, and the
same import failed again.

`Restart=on-failure` is designed for a process that runs correctly and then
dies. It cannot help a process that fails during startup — restarting only
repeats the same failure.

---

## Fix

```bash
sudo cp /opt/lab-api/app.py.bak /opt/lab-api/app.py
head -1 /opt/lab-api/app.py
sudo systemctl reset-failed lab-api
sudo systemctl restart lab-api
```

Recovery was immediate — `active (running)` at 00:38:51, HTTP 200 four seconds
later.

**`reset-failed` was run but was probably not required in this case.** The unit
was still in `activating (auto-restart)` and had not yet hit the start rate
limit. It becomes necessary once the unit enters the hard `failed` state
(see below).

---

## Prevention

- **`activating (auto-restart)` means a loop, not a stable failure.** Read the
  state word carefully: `active` (fine), `failed` (stopped trying),
  `activating (auto-restart)` (currently cycling). Running `systemctl status`
  twice a few seconds apart and watching the PID change confirms it.
- **Do not trust the `Status:` line.** With `Type=notify`, it shows the last
  message the process sent, which can describe a success that has since been
  undone. `Active:` is authoritative; `Status:` is not.
- **systemd stops retrying eventually.** `StartLimitBurst` (default 5) within
  `StartLimitIntervalSec` (default 10s) puts the unit into a hard `failed`
  state where it stops restarting entirely — including across reboots.
  `Restart=on-failure` does not override this. Once there, `systemctl restart`
  alone may refuse; `systemctl reset-failed <unit>` clears the counter first.
  Not knowing this turns a short outage into a long one.
- **Run the application manually before debugging the service manager.**
  `python app.py` in the venv produced a 3-line traceback naming the file and
  line. The journal produced forty lines of gunicorn and importlib frames
  saying the same thing.
- **The useful journal line is usually the last one.** Tracebacks read
  outermost-first; the actual error is at the bottom. `grep -i "error"` reaches
  it faster than scrolling.
- **"Worker failed to boot" is specific.** It means the WSGI module could not
  be imported — a startup problem, not a request-handling problem. That
  distinction determines whether restarting could ever help.
- **Back up before editing a deployed file.** `cp app.py app.py.bak` made
  recovery one command. Note that a file copied with `sudo` is root-owned and
  `rm` will prompt before deleting it.
- **`Restart=on-failure` has no automatic startup-failure detection.** A
  monitoring gap worth closing: a unit stuck in `activating (auto-restart)`
  produces no alert by default. `systemctl is-active` returns `activating`,
  which naive checks may treat as acceptable.
  
