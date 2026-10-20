# Lab 02 — Your First Security Policy (Learning Mode)

**Module 1 · Est. time 75 min · Level: beginner**

## Objectives

- Create an ASM security policy from a Rapid Deployment template.
- Attach it to the virtual server in **transparent** mode.
- Generate attack traffic and read violations + learning suggestions.
- Understand policy build state and traffic learning.

## Prerequisites

Lab 01. `app_vs` serving DVWA.

## Concepts

- **Rapid Deployment template** = "allow everything, log violations" starter policy. Ideal
  for learning mode: the engine starts classifying real traffic immediately.
- **Transparent mode** = violations logged, requests still forwarded. **Blocking** = enforced.
- **Policy build** — ASM builds the policy from learned traffic; status goes
  Unbuilt → Building → Built. Learning suggestions appear as
  **red (new/never seen), amber (seen), green (repeated)** flags.

## Tasks

### 1. Create the policy (GUI)

1. **Security › Application Security › Policies › Policies List › Create**.
2. Name: `dvwa_policy`. Template: **Rapid Deployment Policy**. Application Language: UTF-8.
   Description: `DVWA lab policy`.
3. Leave **learn from traffic** on. Save.

tmsh:

```
tmsh create asm policy dvwa_policy template-name POLICY_TEMPLATE_RAPID_DEPLOYMENT
```

### 2. Attach to the virtual server

```
tmsh modify ltm virtual app_vs profiles add { asm { policy dvwa_policy } }
```

GUI: Local Traffic › Virtual Servers › app_vs › Security › Policies → enable
Application Security → select `dvwa_policy`.

Confirm the shield icon appears next to `app_vs` in the virtual server list.

### 3. Generate baseline (good) traffic

Browse DVWA like a normal user for 2–3 minutes:

- Log in (`admin`/`password`).
- Visit: Home, Instructions, **SQL Injection** page, **XSS (Reflected)** page.
- Submit a normal SQLi form value: `id=1` with Submit.

### 4. Generate attack traffic (logged, not blocked)

```bash
# SQL injection payloads
curl -s "http://10.10.2.100/vulnerabilities/sqli/?id=1%27+or+%271%27%3D%271&Submit=Submit"
# Reflected XSS
curl -s "http://10.10.2.100/vulnerabilities/xss_r/?name=%3Cscript%3Ealert(1)%3C/script%3E"
# Path traversal
curl -s "http://10.10.2.100/vulnerabilities/fi/?page=../../../../../etc/passwd"
# Command injection
curl -s -d "ip=127.0.0.1;cat+/etc/passwd" -d "Submit=Submit" \
  http://10.10.2.100/vulnerabilities/exec/
# Nmap-style scan probe
curl -s "http://10.10.2.100/admin/"
```

Check the responses still returned HTTP 200 with app content — **transparent mode does not
block**.

### 5. Read the results

1. **Security › Event Logs › Application › Requests** — each request lists violations:
   `Attack signature detected`, `Malformed HTTP URL`, `Illegal parameter value`, ...
   Click a request → see the full request, matched signatures with their
   **signature ID / rule / accuracy**, and the **Support ID**.
2. **Security › Application Security › Policy Building › Traffic Learning** —
   suggestions: explicit URLs seen, file types, parameters on the SQLi form
   (`id`, `Submit`), with red/amber/green flags.
3. **Security › Application Security › Policies › dvwa_policy** —
   **Build Status** (rebuild happens on schedule or manually), and
   **Violations list** under Policy Settings.

### 6. Accept your first suggestions

1. Policy Building › Traffic Learning › select suggestions for URLs `/login.php`,
   `/vulnerabilities/sqli/` → **Learn** (accept) them.
2. Re-check the **Entities** tree under the policy: URLs now include those paths.
3. Force a policy build: **Apply Policy** button (top of the policy page).

REST (attach check):

```bash
source lab-env.sh
curl -sk -u "$AUTH" https://$BIGIP/mgmt/tm/asm/policies | grep -o '"name":"[^"]*"'
```

## Verification

- [ ] `app_vs` shows the ASM shield with `dvwa_policy` attached.
- [ ] Attack curls return HTTP 200 (transparent), but
      **Requests log** shows `Attack signature detected` on each.
- [ ] Traffic Learning lists DVWA's URLs/parameters after your browsing.
- [ ] Accepted suggestions appear under the policy's URLs entities.

## Extra credit

- Replay the attack traffic against the **Juice Shop** VIP (add a second VS/policy) and
  compare which violations differ — SPA apps behave differently under the same policy.
- In the Requests log, filter by violation `Attack signature detected` and note the
  signature IDs — you'll reuse them in Lab 03.

## Pitfalls

- **"Policy not attached" error in tmsh** — the profile syntax is
  `profiles add { asm { policy dvwa_policy } }`; `policy` without the `asm` profile name fails.
- **No suggestions after browsing** — learning needs "learn from traffic" enabled (default)
  and traffic that differs from the template; use the browser, not just curl.
- Transparent-mode requests still hit the app — some payloads (e.g., `exec/`) will
  actually execute in DVWA. That's the point of the lab, but don't expose this lab net.

## What you learned

Policy creation from a template, attaching via GUI/tmsh, transparent-mode behavior,
reading the Requests verdict log, and the traffic-learning → suggestion → entities loop.
