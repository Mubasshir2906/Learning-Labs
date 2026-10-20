# Lab 04 — Going Blocking + False-Positive Triage

**Module 1 · Est. time 75 min · Level: intermediate**

## Objectives

- Switch the policy to **blocking** mode safely.
- Learn the standard F5 false-positive triage workflow:
  violation → red flag → learn/relax → enforce.
- Use trusted IPs and staging intentionally.

## Prerequisites

Lab 03. `dvwa_policy` with signature sets enabled; some signatures enforced.

## Concepts

- Switching to blocking is safe **only** when the learned/accepted entities cover your
  app's real traffic and staging holds the risky stuff.
- **Enforcement readiness** (v16+): ASM grades every learned element (red = not ready,
  green = safe to enforce) — the modern replacement for "watch the logs for 2 weeks".
- **Trusted IPs** = source IPs whose requests never get blocked (your desk, scanners) —
  great for testing while you build.

## Tasks

### 1. Baseline: is the app still usable?

Normal-user pass (browser): login, all DVWA pages, form posts. Note anything that breaks.
If something breaks in transparent mode, fix that first — blocking makes it worse.

### 2. Set trusted IP (your workstation)

Policy › `dvwa_policy` › **IP Address › Whitelist** → add `192.168.20.x`? No wait —
the client IP behind BIG-IP is your client's source IP (automap changes what the pool
sees, but ASM sees the client IP — verify in the Requests log, it shows the client IP).

Add your actual client IP as seen in the Requests log → Save. Confirm:
from the trusted IP, an attack curl now **passes through even when enforced**
(trusted IPs bypass blocking entirely).

> Don't leave production trusted IPs in place — this is lab-only convenience.

### 3. Flip to blocking

```
tmsh modify asm policy dvwa_policy policy-settings enforcement-mode blocking
tmsh save sys config
```

GUI: Policy Settings › **Blocking Settings** → toggle
**Application Security → Blocking → Enforce** (v17 UI: Policy › Settings › Blocking).

### 4. Break something on purpose (a false positive)

DVWA's **Brute Force** page (vulnerabilities/brute/) uses a GET parameter `username`,
`password`, `Login`. Craft a benign-but-suspicious value:

```bash
curl -s "http://10.10.2.100/vulnerabilities/brute/?username=admin%27&password=pass&Login=Login"
```

The `'` in `username` trips a SQLi signature → 403. This is a classic false positive for
apps that echo such input, and it demonstrates the triage flow.

### 5. Triage workflow

1. Requests log → the blocked request → note the **violation** and the
   **signature ID** that fired on parameter `username`.
2. Policy › `dvwa_policy` › Attack Signatures → Staged/enforced list → find that
   signature → check "Staging" for this signature **in this specific context** if
   available (per-parameter staging: edit the signature → "Make signature ID X on
   parameter username stage").
3. Re-run the curl → **passes** now (staged = log-only) while the generic SQLi attack
   from Lab 03 still blocks.
4. Alternative triage (entity-based): Policy › Parameters › edit the `username`
   parameter → set **Attack Signatures Check = Disabled** (or per-signature staging) —
   a heavier relaxation; note the trade-off.

### 6. Enforcement readiness pass

1. Policy › **Policy Building › Enforcement Readiness** — review red flags.
2. Pick a red-flag item (e.g., a URL seen once), view its traffic, then
   **Learn → Enforce** it and re-verify normal browsing still works.

### 7. Custom blocking page preview

Policy › Policy Settings › **Blocking Settings** → Response Pages → preview the
default block page; note the Support ID text (Lab 08 customizes this).

## Verification

- [ ] Blocking mode on; attack traffic (Lab 03 payloads) → 403 + Support ID.
- [ ] The deliberate false positive blocked first, then passed after staging
      the exact signature on that parameter.
- [ ] Generic SQLi attack still blocked after the relaxation (check!).
- [ ] Normal browsing unaffected.

## Extra credit

- Add a second client IP (untrusted) and verify blocking applies to it while the trusted
  IP sails through.
- Use Burp Repeater to fire a payload that triggers **two** violations; block on
  `Malformed URL` first, then relax it, and see the signature violation take over.
- Document a triage decision in the policy's **Description** field — audit trails matter.

## Pitfalls

- **Blocking on before entities learned** → you'll 403 your own app's pages. Always
  transparent-learn first (Lab 02), then flip.
- **Staging never expires** — after 7 days staged signatures auto-promote in blocking
  mode; if you wanted them log-only forever, use per-signature "stage" without expiry or
  disable the set.
- **Trusted IP format** — ASM whitelist takes single IPs or CIDR; `10.0.0.1-10.0.0.5`
  ranges are not supported.

## What you learned

The production go-live pattern: learn transparent → review readiness → flip blocking →
triage false positives with surgical staging, not blunt disables.
