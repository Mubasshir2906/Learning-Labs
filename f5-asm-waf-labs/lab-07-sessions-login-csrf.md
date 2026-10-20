# Lab 07 — Sessions: Login Pages, Login Enforcement, Session Tracking, CSRF

**Module 2 · Est. time 90 min · Level: intermediate**

## Objectives

- Configure login pages and **login enforcement** (block access to protected URLs
  without an authenticated session).
- Enable **session tracking** and set a session limit.
- Enable **CSRF protection** with token injection and prove it blocks token-less requests.

## Prerequisites

Lab 06. `dvwa_policy` blocking; parameter model done.

## Concepts

- **Login page** definition tells ASM: "this URL authenticates users; success looks like
  this" (a response containing "Logout", a redirect to a known post-login URL, a session
  cookie set, or a specific status code).
- **Login enforcement** (AWAF): requests to protected URLs without a valid ASM-tracked
  session → violation `Access to disallowed URL`/`Login enforcement` — blocks
  deep-linking with no session.
- **Session tracking**: ASM tracks sessions (login → logout) per user; you can limit
  concurrent sessions, inspect session IDs, and block when limits are exceeded.
- **CSRF protection**: ASM injects a nonce into GET/POST forms on protected URLs and
  validates it on subsequent requests. No token → `CSRF attack detected`.

## Tasks

### 1. Define the login page

Policy › **Sessions and Logins › Login Pages** → Create:

- URL: `/login.php`, method POST, parameters: `username`, `password`.
- **Login response** (what success looks like): response contains string `Logout`.
- Access validation: **start session on login success**.

Apply. Then test login through the VIP:

```bash
curl -s -i -d "username=admin&password=password&Login=Login" http://10.10.2.100/login.php \
  | grep -E "HTTP|Set-Cookie|Location"
```

Note the `PHPSESSID` cookie — ASM now maps this to a tracked session.

### 2. Session tracking

Policy › **Sessions and Logins › Session Tracking** → enable:

- **When disposition is enforce... track sessions** → Yes.
- Session timeout: 10 min (lab-short; production typically 30+).
- **Maximum concurrent open sessions per user**: 2 (to demo the limit).
- Logout URL `/logout.php` → define it as **Logout URL** (ends the tracked session).

Test the limit: log in as admin from two curl instances (separate cookies), then a third:

```bash
# three parallel logins, different cookies each time
for i in 1 2 3; do curl -s -i -c c$i.txt -d "username=admin&password=password&Login=Login" \
  http://10.10.2.100/login.php | head -n 1; done
```

Check Requests log for `Login violation`/session-limit hits on the third.

### 3. Login enforcement (protect deep links)

Policy › **Sessions and Logins › Login Enforcement**:

- Enable; add protected URLs: `/index.php`, `/vulnerabilities/*`, `/security.php`,
  `/instructions.php`, `/logout.php`? (careful: logout usually allowed).
- Entry point: `/login.php`.

Test — request a protected URL with **no session**:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://10.10.2.100/index.php
```

**Expected:** blocked (no tracked session). Then with a valid session cookie from
step 1 → allowed. This kills unauthenticated deep-linking and many scanner classes.

> Login enforcement requires AWAF. On classic ASM the equivalent is
> "URL → requires session" per-entity settings — configure it per-URL instead.

### 4. CSRF protection

Policy › **Sessions and Logins › CSRF Protection**:

- Enable; protected URLs: `/vulnerabilities/sqli/`, `/vulnerabilities/exec/`
  (form-based POSTs), `/login.php` as entry point.
- **CSRF token injection**: GET/POST forms — ASM rewrites responses to add the token.
- Token length default; expiration default (10 min lab).

Test from a browser (tokens need JS/forms rendered):

1. Log in to DVWA, open SQL Injection page, submit `id=1` → works (ASM injected token).
2. Re-fire the same form POST without the token via curl:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  "http://10.10.2.100/vulnerabilities/sqli/?id=1&Submit=Submit"
```

**Expected:** 403 `CSRF attack detected`. Note the Requests log shows the CSRF violation
distinct from parameter/signature violations.

3. Verify browser flows still work end-to-end — CSRF misconfiguration is a top
   false-positive generator; the accept-and-verify loop matters here.

### 5. Watch sessions live

Security › **Event Logs › Application › Sessions** — see tracked sessions: user,
login time, last request, client IP. Kill/expire a session and verify the next request
with that cookie is blocked.

## Verification

- [ ] Login page defined; login success creates a tracked session (Sessions log).
- [ ] Deep-link without session → blocked; with session → allowed.
- [ ] Concurrent session limit fires on the third login.
- [ ] CSRF token-less form POST → 403; browser form flow works.

## Extra credit

- Change "response contains" to a wrong string (e.g., `Welcome`), watch login enforcement
  break (no session ever starts) — then fix. This is the classic login-page misconfig.
- Add `/vulnerabilities/xss_r/` to CSRF-protected URLs and check DVWA's XSS demo still
  works (it's a GET — CSRF on GETs is aggressive; note the trade-off).
- Log out, then replay the old `PHPSESSID` — should be blocked by session tracking
  (invalidated session).

## Pitfalls

- **Login response string must match post-login HTML exactly** — if DVWA's post-login
  page changes (theme switch), enforcement silently breaks.
- **CSRF + SPAs** — DVWA is server-rendered so injection works; for JS apps (Juice Shop)
  token injection needs the JSON/AJAX-aware settings (Lab 13 revisits).
- **Session limit vs NAT** — many users behind one IP look like one user; limit by
  username (ASM does) but know load-testers will trip it.

## What you learned

Login-page modeling, session tracking and limits, login enforcement against unauthenticated
deep links, and CSRF token injection with validation — the session-layer defenses on top
of request-layer checks.
