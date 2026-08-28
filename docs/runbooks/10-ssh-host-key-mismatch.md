# Incident 10 — SSH refuses to connect: host identification changed

**Date:** 2026-08-28
**Layer:** SSH / trust
**Severity:** Complete loss of new SSH access. Existing sessions unaffected.

**Note on method:** genuine IP recycling — where AWS reassigns a public
address previously held by a different instance — cannot be produced on demand.
This incident regenerates the host's ED25519 key instead. The client-side
error, the verification requirement, and the fix are identical; only the
mechanism differs. That distinction is stated explicitly rather than implied.

---

## Symptoms

New SSH connections are refused outright:
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ED25519 key sent by the remote host is
SHA256:tHR+dpou/ggpgmC4bToVkN2sS6Ni/99i79L67DN56t8.
Please contact your system administrator.
Add correct host key in /Users/<user>/.ssh/known_hosts to get rid of this message.
Offending ED25519 key in /Users/<user>/.ssh/known_hosts:5
Host key for 44.243.82.81 has changed and you have requested strict checking.
Host key verification failed.

Sessions established before the change continued working normally.

---

## Investigation

**1. This is not the same as the benign address-change prompt.**

Earlier the same day, after a stop/start assigned a new public IP, SSH produced
a different message:
The authenticity of host '44.243.82.81' can't be established.
ED25519 key fingerprint is: SHA256:2B4V7OMIzGLmJTK6O4+jHiDiOxFZgu2/E//4ACVZK2U
This key is known by the following other names/addresses:
~/.ssh/known_hosts:2: 44.252.44.159
Are you sure you want to continue connecting (yes/no/[fingerprint])?

| Situation | SSH behavior |
|---|---|
| Same key, new address | Notes the other address, asks yes/no |
| Different key, same address | Refuses to connect at all |

The first is a question. The second is a wall. Recognising which one is on
screen determines whether this is routine or a security event.

**2. The warning states the fingerprint being offered.**
`SHA256:tHR+dpou/ggpgmC4bToVkN2sS6Ni/99i79L67DN56t8` — this is the value to
verify against, not to accept on faith.

**3. Verified out-of-band before trusting.**
A session opened *before* the change was still connected, so the host's actual
key could be read directly:
```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:tHR+dpou/ggpgmC4bToVkN2sS6Ni/99i79L67DN56t8 root@ip-10-0-1-65 (ED25519)
```
Match confirmed the host genuinely changed its key and no interception was
occurring.

**This step is the incident.** Clearing known_hosts and reconnecting takes ten
seconds and works whether or not an attacker is present. Verification is what
distinguishes handling the alert from dismissing it.

**4. SSH named the exact offending line.**
`Offending ED25519 key in ... known_hosts:5` — matching the line found earlier
with `grep -n "44.243.82.81" ~/.ssh/known_hosts`.

---

## Commands

Client:
```bash
grep -n "<host-ip>" ~/.ssh/known_hosts
ssh-keygen -R <host-ip>
ssh -i ~/.ssh/lab-key.pem ubuntu@<host-ip>
```

Host:
```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
sudo systemctl restart ssh
sudo ss -tlnp | grep :22
```

---

## Evidence
Original fingerprint SHA256:2B4V7OMIzGLmJTK6O4+jHiDiOxFZgu2/E//4ACVZK2U
After regeneration SHA256:tHR+dpou/ggpgmC4bToVkN2sS6Ni/99i79L67DN56t8
Offered in warning SHA256:tHR+dpou/ggpgmC4bToVkN2sS6Ni/99i79L67DN56t8 ← match
After restore SHA256:2B4V7OMIzGLmJTK6O4+jHiDiOxFZgu2/E//4ACVZK2U

Removal output:
ssh-keygen -R 44.243.82.81

Host 44.243.82.81 found: line 5

/Users/<user>/.ssh/known_hosts updated.
Original contents retained as /Users/<user>/.ssh/known_hosts.old

After restoring the original key, reconnecting produced the *benign* message
rather than the strict-checking refusal:
ED25519 key fingerprint is: SHA256:2B4V7OMIzGLmJTK6O4+jHiDiOxFZgu2/E//4ACVZK2U
This host key is known by the following other names/addresses:
~/.ssh/known_hosts:2: 44.252.44.159
SSH matched the restored key against the entry recorded under the instance's
previous public IP. Same host, same key, different address — a prompt, not a
refusal. Both forms of the message were produced within minutes of each other,
against the same host.

---

## Root cause

The host's ED25519 key pair was regenerated, so the key presented at connection
no longer matched the entry pinned in the client's `known_hosts`. SSH's strict
host key checking refuses the connection rather than prompting.

In production the legitimate causes are: a rebuilt or reimaged host, a
recycled IP now held by a different instance, a restored-from-snapshot server,
or a container or VM rebuilt from a base image. The illegitimate cause is a
man-in-the-middle. **The error alone does not distinguish them** — that is the
entire point of the warning.

---

## Fix

```bash
# client — remove only the stale entry
ssh-keygen -R 44.243.82.81

# reconnect, verifying the fingerprint before accepting
ssh -i ~/.ssh/lab-key.pem ubuntu@44.243.82.81
```

`ssh-keygen -R` removes one host's entry and backs up the previous file to
`known_hosts.old`. Editing `known_hosts` by hand risks corrupting unrelated
entries.

---

## Prevention

- **Verify the fingerprint out-of-band before clearing the entry.** The
  reflexive fix — delete the line, reconnect, accept — is indistinguishable
  from what an attacker would want. Confirming the fingerprint through a
  separate channel is the only step that provides any assurance. Sources: a
  session opened before the change, the EC2 console's **Get system log**
  (which prints host key fingerprints at boot), a configuration management
  record, or a colleague on a different network.
- **Keep an existing session open before touching host keys.** Established
  connections survive; new ones fail. That session is both the recovery path
  and the out-of-band verification channel. Losing it during this change means
  losing access to the host.
- **Distinguish the two SSH messages.** "Known by other names/addresses" with a
  yes/no prompt means the same key appeared at a new address — routine after a
  stop/start. The full `IDENTIFICATION HAS CHANGED` block means a different key
  at a known address — treat as a security event until verified.
- **Use `ssh-keygen -R`, not manual editing.** Targeted, backs up the original,
  cannot corrupt other entries.
- **Recycled public IPs make this routine in cloud environments.** Without an
  Elastic IP, an address released by one instance can be assigned to another
  belonging to someone else entirely. Frequent host key warnings train
  operators to dismiss them, which is precisely how a real interception would
  go unnoticed. Using DNS names rather than raw IPs, or `HostKeyAlias` in
  `~/.ssh/config`, reduces the noise.
- **The EC2 console's system log is the AWS-native verification channel.**
  EC2 → Instances → Actions → Monitor and troubleshoot → Get system log. Host
  key fingerprints are printed during boot. Note the limitation encountered
  here: keys regenerated *after* boot do not appear in that log, so this
  channel works for freshly launched instances but not for post-boot changes.
- **Never use `StrictHostKeyChecking=no` as a general fix.** It suppresses this
  warning entirely and disables the protection it exists to provide. Acceptable
  only for ephemeral, throwaway hosts in automation — never for anything
  persistent.
