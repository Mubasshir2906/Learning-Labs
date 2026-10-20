# Lab 05 — Entities: URLs, File Types, Methods, Headers, Cookies

**Module 2 · Est. time 75 min · Level: intermediate**

## Objectives

- Move from Rapid Deployment "allow-all" to an explicit entity model.
- Enforce allowed file types, HTTP methods, and explicit URLs.
- Use wildcard-URL exceptions correctly.
- Control headers and cookies.

## Prerequisites

Lab 04. `dvwa_policy` in blocking mode.

## Concepts

- **Positive security** starts here: instead of only blocking bad patterns, define what's
  allowed. ASM's entity hierarchy: **File types → URLs → Parameters** (+ headers, cookies,
  methods, redirects).
- Every URL has a length limit, optional regex matching, and its own allowed
  methods/parameters. The **`*` wildcard URL** exists per file type — requests that don't
  match an explicit URL fall to `*`, and if `*` is disallowed → "URL not allowed" violation.
- **File types** filter everything: a request to a disallowed file type (`.exe`, `.bak`)
  is blocked before URL checks.

## Tasks

### 1. Baseline: what does DVWA actually use?

Review what traffic learning gathered (Lab 02): file types `php, css, js, png, gif, ico,
woff2`; methods `GET, POST`; URLs like `/login.php`, `/index.php`,
`/vulnerabilities/*`, `/security.php`, `/logout.php`.

### 2. Tighten file types

Policy › **File Types**:

- Keep allowed: `php, css, js, png, gif, ico, woff2`.
- Note the wildcard `*` file type — requests with unrecognized extensions match it.
- Create a **disallowed** entry for `bak` and `exe`.

Test:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://10.10.2.100/login.php.bak"   # blocked
curl -s -o /dev/null -w "%{http_code}\n" "http://10.10.2.100/login.php"       # 200
```

**Expected:** first → 403 `Illegal file type` (blocked); second → 200.

### 3. Restrict HTTP methods

Policy › **URLs › Allowed methods**: keep GET/POST, block TRACE/DELETE/PUT/PATCH.

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X TRACE http://10.10.2.100/login.php
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE http://10.10.2.100/login.php
```

**Expected:** 403 `Method not allowed` on both (check Requests log for the violation).

### 4. Explicit URLs — make `*` disallowed

The real positive-security move:

1. Policy › **URLs** — ensure every legit DVWA URL is learned and enforced:
   `/login.php`, `/index.php`, `/vulnerabilities/sqli/`, `/vulnerabilities/xss_r/`,
   `/vulnerabilities/fi/`, `/vulnerabilities/exec/`, `/vulnerabilities/brute/`,
   `/security.php`, `/instructions.php`, `/logout.php`, `/about.php`,
   plus statics like `/dvwa/css/*` style paths (learn them from browsing).
2. Set the **`*` wildcard URL** (under file type `*`) to **Disallowed**.
3. Apply policy, then test an unlisted URL:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "http://10.10.2.100/adminer.php"   # 403
curl -s -o /dev/null -w "%{http_code}\n" "http://10.10.2.100/login.php"     # 200
```

**Expected:** unlisted path → 403 `URL not allowed`.

4. Now browse DVWA fully in a normal browser. Anything 403ing = missed entity; accept the
   suggestion or add the URL manually, repeat until clean. **This loop is the core skill.**

### 5. URL detail settings

Edit `/vulnerabilities/sqli/`:

- **Method**: GET only (that page never POSTs).
- **Length**: set URL length 128 (default 8192 is loose).
- **Perform Staging**: on for anything newly added.

Test a POST to a GET-only URL → 403 `Method not allowed`.

### 6. Headers & cookies

Policy › **Headers**:

- Note `Host`, `User-Agent`, `Referer` etc. appear from learning with wildcard `*` header
  as allowed (default: any header name allowed, with per-header value checks).
- Add an explicit **mandatory** header requirement: edit `Host` header entity →
  mandatory = yes. Then send a Host-less request:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Host:" http://10.10.2.100/login.php
```

- **Cookies**: Policy › Cookies — DVWA's `PHPSESSID` and `security` cookies; set
  `security` cookie as **mandatory** on the sqli URL (the app sets it) and test a request
  without it → violation `Missing mandatory cookie`? (Verify behavior; not all contexts
  enforce it the same — observe and note.)

### 7. Redirects

Policy › **Redirections**: the app redirects to `/login.php` from protected pages when
unauthenticated. Ensure redirect domains list contains your VIP/host, and note
"maximum redirect depth". Test an open-redirect-style path if DVWA has one
(`logout.php` then back) — just observe which violations, if any, fire.

## Verification

- [ ] `.bak`/`.exe` file types → 403.
- [ ] TRACE/DELETE → 403; POST on GET-only URL → 403.
- [ ] Unlisted URL with `*` disallowed → 403 `URL not allowed`, legit URLs → 200.
- [ ] Full normal browser session has **zero** blocks after fixing entities.

## Extra credit

- Use URL **regex matching** (`^/vulnerabilities/[a-z_]+/$`) for the vulnerabilities
  section instead of listing each — cleaner for apps with dynamic routes. DVWA's trailing
  paths make a good exercise.
- Find the **`Illegal request length`** violation by sending a 10 KB URL, then set sane
  per-URL lengths so it blocks only where you choose.

## Pitfalls

- **Making `*` disallowed too early** = 403 storm. Every legit URL/asset (favicon, fonts,
  API routes) must be explicitly present first.
- **Static assets under different extensions** (e.g., `.map`, `.woff2`) — browsers load
  them late (devtools open), so your transparent-phase browsing must include devtools open.
- Deleting the `*` URL/file-type wildcard entirely is possible but changes default-match
  semantics — keep it, set disposition instead.

## What you learned

The entity model behind ASM positive security: file types → methods → explicit URLs →
per-URL method/length rules → header/cookie controls, plus the accept-suggestions loop
that keeps real traffic working.
