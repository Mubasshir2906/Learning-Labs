# Lab 12 — Brute Force & Credential Stuffing Defense

**Module 3 · Est. time 60 min · Level: advanced · AWAF**

## Objectives

- Configure **Brute Force Prevention** on the login page (rate-based and
  acceleration-based detection).
- Simulate a brute-force attack and observe login-violation mitigation.
- Understand **Credential Stuffing Defense** (anti-bot service) vs local brute force.

## Prerequisites

Lab 07 (login page defined, session tracking on), Lab 11 (bot defense basics).

## Concepts

- **Brute Force Prevention** (Security › Application Security › Brute Force Attacks):
  per-login-URL protection with two detection modes:
  - **Rate-based**: N failed logins per window → block the source for X seconds.
  - **Acceleration-based (ML)**: learns normal login timing, flags deviations.
- Successful-logins-from-many-IPs = **credential stuffing** signature — full defense uses
  the F5 anti-bot service (device ID/reputation) or bot defense + rate limits combined.
- Login violations surface in **Login Violations** (Requests log) and the brute-force
  dashboard.

## Tasks

### 1. Configure brute force prevention

Security › Application Security › **Brute Force Attacks › Prevention**:

- Add: URL `/login.php`, parameters `username`, `password` (from Lab 07 login page).
- Mode: **rate-based**. Failed attempts threshold: 5 per 60 s. Action: **block 300 s**.
- Track by: client IP (default).

Apply. Baseline: one normal login still works.

### 2. Brute-force it

```bash
# 10 wrong logins fast, from one IP
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code} " \
    -d "username=admin&password=wrong$i&Login=Login" http://10.10.2.100/login.php
done; echo
# then correct login -> should ALSO be blocked now (same IP)
curl -s -o /dev/null -w "%{http_code}\n" \
  -d "username=admin&password=password&Login=Login" http://10.10.2.100/login.php
```

**Expected:** failures pass 5 → subsequent requests (even the correct one) blocked for
300 s; Requests log shows **Login violation: Brute force attack detected**.

### 3. Reset and inspect

Security › Application Security › **Brute Force Attacks › Users/Blocked** — see the
blocked IP with expiry. Remove/unblock it, verify login works again.

### 4. Distributed attempt (what stuffing looks like)

If you have a second client IP (another VM/phone hotspot/secondary Docker net):
split the 10 attempts across two IPs at 6 each — with per-IP rate limits neither trips
globally; observe (this motivates anti-bot/device-ID defenses).

### 5. Layered defense (the production pattern)

Combine, and test each layer:

1. **Login enforcement** (Lab 07) — no session → protected URLs blocked.
2. **Parameter validation** on `username` (Lab 06) — `'` payloads die early.
3. **Bot defense** (Lab 11) — `curl` UA blocked before even hitting the form.
4. **Brute force rate-based** (this lab) — repeats blocked.
5. Optional **anti-bot CAPTCHA** on login page if licensed.

Run the sqlmap auth-bypass attempt against the login and watch which layer catches it:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -d "username=admin%27+or+%271%27%3D%271&password=x&Login=Login" \
  http://10.10.2.100/login.php
```

**Expected:** parameter validation blocks it (403) **before** brute force even counts it.

### 6. AWAF Credential Stuffing Defense (read/config)

If licensed + online (System › License shows anti-bot): DoS/Bot profile →
**Credential Stuffing Defense** → enable on login page, mode "track successful logins
across many IPs". Configure, observe the dashboard; otherwise document the flow:
device-ID collection → F5 cloud reputation → mitigation (CAPTCHA/block) on stuffing
patterns (same password+username pairs rotating IPs).

## Verification

- [ ] 5+ failed logins → subsequent attempts blocked; block expires.
- [ ] Blocked IP visible in the brute-force UI; unblock works.
- [ ] SQLi-style login payload blocked by parameter validation (earlier layer).
- [ ] You can articulate: brute force (one IP many tries) vs stuffing (many IPs,
      known-password pairs) and which ASM features address each.

## Extra credit

- Switch to **acceleration-based** detection, re-run the slow-but-odd pattern
  (one login every 3 s for 5 min) and compare detection sensitivity/false positives.
- Add a second login URL (DVWA only has one; simulate with Juice Shop's REST login
  `/rest/user/login`) — note API logins need JSON profile (Lab 13) first.

## Pitfalls

- **Rate-based blocks the correct user too** — one IP doing 5 bad attempts (fat-fingered
  human) locks them out for the window; tune thresholds or add CAPTCHA instead.
- **Forgot unblock** during testing → confusing later labs; always clear blocks before
  moving on.
- Counting **failed** vs **all** logins: ensure "failed attempts" tracking is on
  (configurable) or successful logins inflate the counter.

## What you learned

Login-level protections: rate-based and acceleration-based brute force prevention,
the stuffing threat model, and how session/login/parameter/bot defenses layer.
