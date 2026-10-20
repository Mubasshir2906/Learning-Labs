# Lab 08 — Data Protection: Data Guard, Masking, Custom Blocking Pages

**Module 2 · Est. time 60 min · Level: beginner/intermediate**

## Objectives

- Enable **Data Guard** to block leakage of sensitive data in responses.
- Mask sensitive parameters/cookies in logs and event logs.
- Build a custom blocking response page.
- Apply per-URL response scrubbing.

## Prerequisites

Lab 07. `dvwa_policy` blocking; a browser session to DVWA works.

## Concepts

- **Data Guard** scans **responses** for sensitive patterns (credit cards, SSNs, custom
  regex) and blocks/scrubs them. Policy Settings › Data Guard. Options include checking
  responses only on certain URLs and excluding file types (images, js).
- **Parameter masking** replaces sensitive values with `***` in the Requests log — so
  passwords never sit in logs.
- **Custom response pages** replace the default F5 block page (per-policy or global),
  useful for branding and support workflows (support ID stays!).

## Tasks

### 1. Seed some "sensitive data" in the app

DVWA doesn't leak PII natively. Use the **Command Injection** or **File Inclusion** page
to render a fake card number, or simply create a static page on the app:

```bash
# On the DVWA container:
docker exec dvwa sh -c 'echo "Customer card: 4111 1111 1111 1111, SSN: 123-45-6789" \
  > /var/www/html/leak.html'
```

Baseline (Data Guard off):

```bash
curl -s http://10.10.2.100/leak.html
# → returns the raw numbers
```

### 2. Enable Data Guard

Policy › Policy Settings › **Data Guard**:

- Enable **Data Guard enforcement**.
- Pattern sets: **Credit card numbers**, **U.S. Social Security Numbers**, plus a
  **custom pattern** (see step 4).
- Check response on: all URLs. Exclude file types: `gif,jpg,png,css,js,ico` (perf).

Apply. Test:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://10.10.2.100/leak.html
```

**Expected:** 403 (or scrubbed content, per your "Enforcement" choice). Requests log shows
`Data Guard: HTTP protocol compliance`? — the violation name is
**`Data Guard: HTTP protocol enforcements`** / `Data Guard` detection in the log; note
exactly which violation name your version reports.

### 3. Custom Data Guard pattern

Policy Settings › Data Guard › **Custom patterns** → add regex for lab-internal
"account numbers": `\bACCT-[0-9]{6}\b`.

```bash
docker exec dvwa sh -c 'echo "ACCT-555001" >> /var/www/html/leak.html'
curl -s -o /dev/null -w "%{http_code}\n" http://10.10.2.100/leak.html
```

**Expected:** blocked/scrubbed on the custom pattern too.

### 4. Mask sensitive parameters

Policy › Parameters → edit `password` on `/login.php` → enable
**"Parameter value is sensitive" / Mask in logs** (v17: check the parameter's
"Masked" option).

Fire a login, then check Requests log: the `password` value shows masked, while
`username=admin` is visible.

Mask the **`security` cookie** too (Policy › Cookies → security → mask).

### 5. Custom blocking page

1. Policy Settings › Blocking Settings › **Response Pages** → enable
   "Enable customization of the default response page" → **HTML response page**.
2. Use this minimal branded page (keep the Support ID token):

```html
<html><head><title>Request Rejected</title></head>
<body>
  <h1>Your request was blocked</h1>
  <p>This site is protected by a Web Application Firewall.</p>
  <p>Support ID: ${support_id}</p>
</body></html>
```

3. Apply, fire an attack:

```bash
curl -s "http://10.10.2.100/vulnerabilities/sqli/?id=1%27+or+%271%27%3D%271&Submit=Submit" \
  | grep -i "support id"
```

**Expected:** your custom page renders with a real Support ID. Search that ID in the
Requests log — that's the operational support workflow (user sends you the ID, you find
the exact request).

### 6. Per-URL response scrubbing (AWAF)

If available on your version: Policy › URLs → edit `/leak.html`-equivalent →
"Apply response scrubbing" — compare behavior vs global Data Guard. Note when you'd use
per-URL scrub vs full block.

## Verification

- [ ] Card/SSN/custom-pattern leakage → blocked or scrubbed.
- [ ] Images/js excluded from Data Guard (fast, no false positives there).
- [ ] `password` masked in Requests log; `username` visible.
- [ ] Custom block page shows with Support ID; ID searchable in the log.

## Extra credit

- Add a false-positive test: a page containing the *string* `4111-1111-1111-1111` inside
  a tutorial (like this lab!) — observe Data Guard firing on content, then scope Data
  Guard to exclude that URL. Decide: block, scrub, or exclude?
- Enable "Data Guard framing" (strip content in frames) and see what it does to DVWA's
  iframes (there is one on the setup page).

## Pitfalls

- **Data Guard + big responses** = CPU cost. Always exclude binary/media file types.
- **Scrub vs block** — "Strip content" mode silently alters responses; test with the app
  owner, silent modification can break integrations.
- Masking affects **logs only** — the app still sees real values (by design).

## What you learned

Response-side security (Data Guard), log hygiene via masking, and the operational
block-page workflow anchored on Support IDs.
