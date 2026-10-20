# Lab 03 — Attack Signatures: Sets, Accuracy, Staging

**Module 1 · Est. time 75 min · Level: beginner/intermediate**

## Objectives

- Enable signature sets and understand accuracy/staging.
- Inspect how signatures match real payloads.
- Enforce staged signatures and watch blocks happen.

## Prerequisites

Lab 02. `dvwa_policy` attached, transparent mode, DVWA traffic flowing.

## Concepts

- **Signature system sets** ship in packages; policy enables them via **sets**:
  `Attack Signatures`, `CVE`, `Generic Detection Signatures` (by accuracy High/Medium/Low).
- **Accuracy** = false-positive risk. High-accuracy signatures are safe to auto-enforce;
  Low-accuracy need triage.
- **Staging** = newly added signatures (or learned ones) sit in staging (default 7 days):
  they **log** violations even in blocking mode, but don't block, until promoted.
- Signatures match a specific part (URI, parameter, header, cookie, body) — that's why the
  same payload can trigger different signatures per location.

## Tasks

### 1. Inspect available signature sets

```
Security › Options › Application Security › Advanced Configuration ›
  Attack Signatures › Signature Sets
```

- Note the built-in sets: **Attack Signatures** (system), **CVE Signatures**,
  **Generic Detection (High/Medium/Low)**, **Web Scraping Detection**, etc.
- View the signature list (Security › ... › Attack Signatures › Active Signatures):
  search `SQL` — note `2000000024` (SQL injection generic) and friends. Record 2–3 IDs.

### 2. Enable sets on the policy

GUI: Policies › `dvwa_policy` › **Attack Signatures** → Configure Signature Sets:

- Enable **Attack Signatures** (system set).
- Enable **Generic Detection Signatures (High)** and **(Medium)**.
- Enable **CVE Signatures**.

tmsh:

```
tmsh modify asm policy dvwa_policy signature-sets add { \
  { signature-set { name "Attack Signatures" } } \
  { signature-set { name "Generic Detection Signatures (High)" } } }
```

(Use `tmsh list asm policy dvwa_policy signature-sets` first to see the exact structure.)

Apply the policy.

### 3. Fire signature-matching payloads

```bash
source lab-env.sh
# SQLi — multiple signatures per request
curl -s "http://10.10.2.100/vulnerabilities/sqli/?id=1%20OR%201=1--&Submit=Submit"
# XSS variants
curl -s "http://10.10.2.100/vulnerabilities/xss_r/?name=%3Csvg/onload=alert(1)%3E"
# Known scan probes (CVE-set matches)
curl -s "http://10.10.2.100/phpmyadmin/index.php"
curl -s "http://10.10.2.100/.env"
curl -s "http://10.10.2.100/wp-admin/setup-config.php"
```

### 4. Read which signatures matched

**Security › Event Logs › Application › Requests** → click a request →
**Attack signature detected** details:

- Signature ID, rule text, accuracy, and **which part matched** (e.g., the `id` parameter).
- Note the violations are **logged** (staging/transparent) — requests still succeeded.

### 5. Test sqlmap (noisy scanner)

```bash
# Use a cookie if DVWA requires a session:
# log in via curl, save cookies:
curl -s -c cookies.txt -d "username=admin&password=password" http://10.10.2.100/login.php
sqlmap -u "http://10.10.2.100/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="security=low; PHPSESSID=$(grep -oP 'PHPSESSID\t\K.*' cookies.txt)" \
  --batch --level=1 --risk=1 | tail -n 20
```

In the Requests log you'll see dozens of signature hits — signature detection alone
catches most of it even in transparent mode.

### 6. Promote staged signatures → enforce

1. Policy › `dvwa_policy` › **Attack Signatures › Staged Signatures** —
   see the signatures that fired, sitting in staging.
2. Select all → **Remove from Staging** (= enforce them).
3. **Apply Policy**.
4. Replay the Lab 3.3 curls.

**Expected:** `HTTP/1.1 403` + F5 blocking page with a **Support ID** for the
SQLi/XSS requests; the phpMyAdmin/.env probes blocked too. The Requests log shows
disposition **Blocked**.

tmsh (list staged):

```
tmsh show asm policy dvwa_policy staged-signatures
```

REST:

```bash
source lab-env.sh
curl -sk -u "$AUTH" https://$BIGIP/mgmt/tm/asm/policies/\
~Common~dvwa_policy/staged-signatures '?$select=name,enforcementState' | head -c 2000
```

## Verification

- [ ] Signature sets (Attack, Generic High/Medium, CVE) enabled on the policy.
- [ ] Requests log shows signature IDs + matched parts for each payload.
- [ ] After removing from staging, replayed attacks get **blocked** (403 + Support ID).
- [ ] Normal browsing (Lab 02 baseline) still succeeds — no false positives.

## Extra credit

- Run `sqlmap` again **after** enforcing and watch it get blocked early — note how far
  (which request number) it gets vs the transparent run.
- Find one **Low-accuracy** signature that matches benign traffic on your app (e.g.,
  a parameter containing `<` or `--`) and predict whether promoting it would false-positive.
  Test it, then restore from staging.

## Pitfalls

- **Set enable errors in tmsh** — signature-set names contain parentheses; quote them
  exactly, or use the GUI.
- **Blocking page instead of app 404s** — you enforced `.env`/`phpmyadmin` probes; that's
  correct behavior, not a misconfig.
- Promoting **all** staged signatures after aggressive scanning can false-positive your
  own app (e.g., Generic Low). Promote High/Medium first (Lab 04 covers triage properly).

## What you learned

Signature sets vs individual signatures, accuracy tiers, staging as a safety valve,
reading matched-part details, and promoting staged signatures to go from log-only to block.
