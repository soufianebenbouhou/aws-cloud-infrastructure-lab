# Incident 06 — Service running and healthy locally, refused from the network

**Date:** 2026-08-28
**Layer:** Application / socket bind address
**Severity:** Full external outage of the API. Service reports healthy by every
host-side measure.

---

## Symptoms

- Requests from outside the host fail **immediately** with `Connection refused`
- `systemctl status lab-api` shows `active (running)`
- `curl localhost:5000/health` returns 200 on the instance
- No errors in the application journal
- All AWS network configuration unchanged and correct
- from outside, using the public IP

curl -v --max-time 10 http://<public-ip>:5000/health

Trying <public-ip>:5000...
connect to <public-ip> port 5000 failed: Connection refused
Failed to connect after 0 ms
curl: (7) Failed to connect to <public-ip> port 5000
on the same host, same minute, using loopback

curl -s -w "\nHTTP %{http_code}\n" http://localhost:5000/health
{"status":"ok","time":"2026-08-28T00:34:44.113290+00:00"}
HTTP 200

---

## Investigation

**1. `Connection refused`, not a timeout — and in 0 ms.**
This is the fork that decides everything else.

| Failure | Meaning | Where to look |
|---|---|---|
| Timeout, no response | packet silently dropped | security group, NACL, routing |
| `Connection refused` (RST) | packet arrived, nothing listening | the host |

A refusal means the SYN reached the kernel and the kernel replied with a TCP
RST. Security groups and NACLs drop packets; they never send RST. The 0 ms
timing confirms a local response rather than anything traversing the network.

Incidents 01, 02 and 03 all produced timeouts. This one did not, which
eliminated all three of those causes before opening the console.

**2. The service was healthy.**
`systemctl status` reported active (running). `curl localhost:5000/health`
returned 200 on the instance. The application was working correctly.

**3. `ss` identified the fault in one command.**

Broken:
sudo ss -tlnp | grep 5000
LISTEN 0 2048 127.0.0.1:5000 0.0.0.0:* users:(("gunicorn",pid=4241,...))

Working:
LISTEN 0 2048 0.0.0.0:5000 0.0.0.0:* users:(("gunicorn",pid=4402,...))

`127.0.0.1:5000` binds the loopback interface only. Traffic arriving on `ens5`
has no socket to reach.

**4. Reproduced the split from a single host.**
Running both requests from the instance itself demonstrated the mechanism
without involving the network at all:
curl http://<public-ip>:5000/health → Connection refused
curl http://localhost:5000/health → HTTP 200
Same process, same second. The public IP leaves and re-enters via `ens5`;
localhost stays on `lo`. Only the destination address differs.

**5. AWS network configuration verified unchanged.**
Security group port 5000 open to 0.0.0.0/0; NACL rule 100 ALLOW in both
directions; route table with the IGW route. All correct.

---

## Commands

```bash
sudo ss -tlnp | grep 5000
grep ExecStart /etc/systemd/system/lab-api.service
sudo systemctl status lab-api --no-pager
curl -s -w "\nHTTP %{http_code}\n" http://localhost:5000/health
curl -v --max-time 10 http://<public-ip>:5000/health
```

---

## Evidence

Unit file at time of failure:
ExecStart=/opt/lab-api/venv/bin/gunicorn --bind 127.0.0.1:5000 --workers 2 app:app

Timeline:
00:34:39 bound 127.0.0.1:5000 public IP → Connection refused (0 ms)
00:34:44 bound 127.0.0.1:5000 localhost → HTTP 200
00:35:10 bound 0.0.0.0:5000 public IP → HTTP 200

---

## Root cause

gunicorn was bound to `127.0.0.1:5000` instead of `0.0.0.0:5000`.

`127.0.0.1` is the loopback interface — reachable only from the host itself.
`0.0.0.0` means all interfaces, including `ens5`, which carries traffic from
the VPC and the internet.

The process was running correctly and the port was listening. It was listening
on an interface no external traffic arrives on.

---

## Fix

```bash
sudo sed -i 's|--bind 127.0.0.1:5000|--bind 0.0.0.0:5000|' \
  /etc/systemd/system/lab-api.service
sudo systemctl daemon-reload
sudo systemctl restart lab-api
sudo ss -tlnp | grep 5000
```

`daemon-reload` is required. systemd caches unit files in memory; restarting
without reloading applies the previously cached definition and the edit appears
to have no effect. This is a common cause of "I changed the config and nothing
happened."

---

## Prevention

- **Read the failure verb first.** `Connection refused` and a timeout are
  different diagnoses. Refused means the host answered — look at the process
  and the bind address. Timeout means something dropped the packet — look at
  security groups, NACLs, and routes. This single distinction eliminated three
  previously documented root causes without opening a console.
- **`ss -tlnp` is the definitive check for "is it listening where I think."**
  `systemctl status` reports the process, not the socket. A service can be
  active and running while listening on the wrong interface.
- **Test from the target network, not just from the host.** `curl localhost`
  returns 200 in this failure state. A deployment check that only tests
  loopback passes while the service is externally unreachable.
- **Testing both addresses from one host isolates the layer instantly.**
  `curl <public-ip>` versus `curl localhost` on the same machine gives the
  answer without touching AWS.
- **Loopback binding is often a default, not a mistake.** Flask's development
  server, many database servers, and several application frameworks bind
  `127.0.0.1` by default as a safety measure. Deploying with an unchanged
  default produces this exact failure.
- **Verify the bind address after any unit file change.** The value lives in
  `ExecStart` and is easy to alter without noticing. `ss -tlnp` after every
  restart confirms what actually happened rather than what was intended.
- **VPC Flow Logs would show ACCEPT here.** The packets were delivered — the
  VPC did its job. Flow logs showing ACCEPT while the client reports failure is
  the signal that the fault is above the network layer, and distinguishes this
  from Incident 02 where the reply was REJECTed.
  
