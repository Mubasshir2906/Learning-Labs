# Lab 10 — L7 DoS Protection & Rate Limiting

**Module 3 · Est. time 75 min · Level: intermediate**

## Objectives

- Create a DoS profile with per-client-IP **request limits** and per-URL TPS limits.
- Attach it to the virtual server and verify rate limiting blocks floods.
- Use the DoS dashboard and `Rate Limit` violations.

## Prerequisites

Lab 09. Load generator `hey` installed (Lab 00).

## Concepts

- **DoS profile** (Security › DoS Protection) attaches to a VS like a policy. Two flavors:
  - **Legacy/L7 DoS**: static rate limits — "N requests per M seconds per source IP",
    optionally per URL.
  - **Behavioral (BDP)**: machine-learned baselines — Lab 11.
- Violations land in the **DoS Dashboard** (Security › DoS Protection › Dashboard),
  and blocked requests appear in the Requests log with `Rate limit` disposition.

## Tasks

### 1. Create the DoS profile

Security › **DoS Protection › DoS Profiles** → Create:

- Name: `dvwa_dos_profile`. Type: **Application Security** (L7).
- Under **Rate Limiting / Application** (v17: "Behavioral & Stress-based Detection"
  off for now — that's Lab 11):
  - **Requests limit**: enable, 20 requests / 10 sec, per **client IP**.
  - Action: **block** (and note alternatives: `rate limit` HTTP code 429, captcha).
- Add a **per-URL rule**: `/vulnerabilities/exec/` limit 5 req / 10 sec (form POSTs
  should be rarer).

tmsh (structure — names differ slightly per version; inspect with
`tmsh list security dos profile` after GUI creation and diff):

```
tmsh create security dos profile dvwa_dos_profile application add { app { \
  dos-allowed-request-limits { client 20 per 10 } } }
```

### 2. Attach to the virtual server

```
tmsh modify ltm virtual app_vs security add { dos-profiles { dvwa_dos_profile } }
```

GUI: app_vs › Security › Policies → DoS Profile → `dvwa_dos_profile`.

### 3. Baseline sanity

Normal browsing still works (you're nowhere near 20 req/10s as a human).

### 4. Flood it

```bash
hey -n 200 -c 5 -z 30s http://10.10.2.100/login.php
```

Watch the status histogram live; then:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://10.10.2.100/login.php
```

**Expected:** during/after the flood your IP trips the limit → 403 (or 429 if you chose
rate-limit action) with `Rate limit` in the log. After the window, access returns.

### 5. Per-URL limit

```bash
hey -n 60 -c 2 -z 15s "http://10.10.2.100/vulnerabilities/exec/"
```

**Expected:** the exec URL trips at ~5/10s even though total site traffic is lower than
the global limit.

### 6. Read the dashboards

- **Security › DoS Protection › Dashboard** — attack events, mitigated IPs graph,
  "DoS Application" attack type `Request Limit`.
- **Security › Event Logs › Application › Requests** — filter by
  `Request event → Rate limit`.
- Note: DoS blocks are **not** policy violations — they're profile-level. Different log
  view, different tuning knob.

### 7. Whitelist a trusted source

DoS profile › Application → **Allowlisted IPs** — add your monitoring host (simulate
with a second client IP if you have one) and verify it doesn't get rate limited during
a flood.

## Verification

- [ ] Flood from `hey` → blocked with Rate limit violation; dashboard shows the attack.
- [ ] Per-URL limit fires before the global limit on `/vulnerabilities/exec/`.
- [ ] Normal browsing unaffected; allowlisted IP unaffected.
- [ ] After the window, requests flow again (no permanent block).

## Extra credit

- Change the action from **block** to **HTTP 429** with `Retry-After` and observe
  client behavior differences.
- Enable **TPS-based alerting**: profile → alarm limits — set DoS alert threshold and
  see the Dashboard raise an "alarm" event (look in DoS logs).
- Compare `tmsh show sys performance` CPU during flood with profile attached vs detached.

## Pitfalls

- **20 req/10s per IP can block a busy office NAT** — real deployments need
  per-IP whitelists or BDP instead of static limits for shared egress.
- **Rate limits don't deduplicate by user** — one attacker with a botnet spreads across
  IPs; that's why BDP exists (Lab 11).
- VS must have the **DoS profile in `security` section**, not `profiles` — common tmsh
  mistake (`security add { dos-profiles { ... } }`).

## What you learned

Static L7 DoS: per-IP and per-URL rate limits, actions and codes, dashboards and
log locations, and where static rate limiting falls short.
