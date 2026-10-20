# Lab 15 — Logging, Remote Logging & SIEM

**Module 4 · Est. time 75 min · Level: intermediate**

## Objectives

- Control what ASM logs (log profiles, per-policy logging settings).
- Configure **remote logging** (high-speed logging → syslog publisher → SIEM).
- Run a local SIEM stand-in (rsyslog/Python) and parse CEF/ASM events.
- Use logs for forensics: reconstruct an attack from raw events.

## Prerequisites

Lab 14. Blocking policy running; an attack surface to generate events.

## Concepts

- **Storage filter / log profile**: what gets logged per request (all, violations only,
  blocked only) and where (local + remote).
- **Remote logging** chain on BIG-IP:
  `Log Profile → Remote Storage (Publisher) → HSL destination (syslog pool/remote host)`.
- ASM events reach the SIEM as **syslog CEF/LEEF** lines (Splunk CEF, QRadar LEEF)
  or via the **Logging API** / **Telemetry Streaming** for JSON pipelines.

## Tasks

### 1. Tune the log profile

Security › Event Logs › **Logs › Create filter**:

- Scope: application security events, all requests for this lab (production: violations).
- Storage: local; note `Security › Event Logs › Application › Requests` keeps
  max 20k requests (default) — tune/understand retention.

CLI view of local log: on the BIG-IP shell (advanced → `bash`):

```bash
tail -n 20 /var/log/asm
```

### 2. SIEM stand-in (Docker rsyslog)

On the Docker host:

```bash
docker run -d --name siem -p 514:514/udp -p 514:514/tcp \
  -v siem-logs:/var/log rsyslog/syslog? # if unavailable, use any rsyslog image, or:
# simplest no-image option: tiny python collector
python3 - <<'EOF'
import socket, datetime
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.bind(("0.0.0.0", 514))
while True:
    data, addr = s.recvfrom(65535)
    with open("asm-siem.log", "a") as f:
        f.write(datetime.datetime.now().isoformat() + " " + data.decode(errors="replace") + "\n")
EOF
```

Note the Docker host IP reachable from the BIG-IP (e.g., `192.168.10.20`).

### 3. Remote logging objects on BIG-IP

```
tmsh create sys syslog log-remote-syslog? # version-dependent; GUI path is easier:
```

GUI: System › Logs › Configuration › **Remote Logging → Syslog** (or
Security › Options › Application Security › Advanced Configuration › Remote Logging):

1. **Remote Logging → Syslog**: add remote host `192.168.10.20:514` (your SIEM stand-in),
   protocol UDP, format: **ASM logs** (or `user log`).
2. Security › Event Logs › Logs › **Log Profile** → Remote storage →
   use the **publisher** referencing that syslog target.

(Exact object names: `tmsh create security log profile ... remote-storage ... publishers`
— create via GUI, then `tmsh list security log profile` to see the full JSON structure.)

### 4. Generate events, watch them flow

```bash
curl -s -o /dev/null "http://10.10.2.100/vulnerabilities/sqli/?id=1%27+OR+%271%27%3D%271&Submit=Submit"
curl -s -o /dev/null -A "curl/8.0" http://10.10.2.100/login.php          # bot block
hey -n 100 -c 10 http://10.10.2.100/login.php                            # rate limit
```

Check your SIEM stand-in file / console: ASM event lines arrive — parse one fully:
client IP, method, URI, violation(s), signature IDs, request status, support ID.

### 5. Forensics drill

Reconstruct from logs only:

1. Which attacks hit today, by violation type? (grep/aggregate the log).
2. Which signatures fired most? Any repeated source IP?
3. Find the request by its **Support ID** and print the full record.

```bash
grep -c "SQL" asm-siem.log        # etc.
```

### 6. (Optional) Telemetry Streaming to JSON

If you want the modern pipeline: install **F5 Telemetry Streaming** (iApp/LX) and configure
a consumer (generic HTTP to your SIEM, or Splunk HEC). Compare JSON TS events vs classic
syslog — TS gives structured JSON that ELK/Splunk ingest cleanly. Full TS setup is a
lab of its own; do a minimal consumer pointing at your Python collector.

## Verification

- [ ] Requests log + `/var/log/asm` both show events locally.
- [ ] Remote syslog events arrive at the stand-in SIEM.
- [ ] Forensics drill answers: top violation, top signature, attack sources — from logs.

## Extra credit

- Switch remote logging to **TCP** and compare loss under flood (UDP drops).
- Send events in **Splunk CEF** vs plain and note which fields differ.
- Alert rule idea: implement "alert if blocked requests from one IP > 10 in 1 min" as a
  grep loop on the stand-in — you've built a mini-SIEM correlation.

## Pitfalls

- **UDP + floods** = lost events exactly when you need them; production: TCP with
  buffering, or F5 Cloud Logging/TS.
- **Log profile granularity** — logging *all* requests in production is heavy; log
  violations by default, all-requests only during tuning windows.
- BIG-IP needs a route to the SIEM (mgmt route or TMM route — with **management-port
  logging** use `sys syslog include` filters carefully; prefer TMM-side publisher for
  high-volume).

## What you learned

The full logging chain (profile → publisher → syslog), local vs remote storage, parsing
ASM events, and using logs for attack forensics — the operational half of WAF ownership.
