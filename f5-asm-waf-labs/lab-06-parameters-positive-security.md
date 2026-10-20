# Lab 06 — Parameters & Positive Security Validation

**Module 2 · Est. time 90 min · Level: intermediate**

## Objectives

- Configure parameter validation: value types, data types, lengths, patterns.
- Use parameters to block injection attacks **without** signatures.
- Accept parameter suggestions from learning.
- Handle JSON parameters.

## Prerequisites

Lab 05. `dvwa_policy` blocking, explicit URL model working.

## Concepts

- ASM validates each request **parameter** against its definition:
  - **Value type**: `user-input` (attacker-controlled), `dynamic-content` (server sets),
    `static-content` (constant).
  - **Data type**: alpha, alphanumeric, decimal, integer, email, boolean, ... plus
    **min/max length** and **regex pattern**.
  - Optional per-parameter **Attack Signatures check** on/off and per-parameter staging.
- This is the **positive security model** at its best: `id` must be `^[0-9]{1,4}$`
  → `' OR '1'='1` is impossible, signatures or not.

## Tasks

### 1. Discover the parameters

From traffic learning (Lab 02) and the Requests log, DVWA has:

- `/login.php` (POST): `username`, `password`, `Login` (button)
- `/vulnerabilities/sqli/` (GET): `id`, `Submit`
- `/vulnerabilities/exec/` (POST): `ip`, `Submit`
- `/vulnerabilities/xss_r/` (GET): `name`

Confirm in Policy › **Parameters** — the learned ones sit there with wildcard `*`
parameter (value type user-input) as the fallback.

### 2. First: signatures block it (baseline)

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  "http://10.10.2.100/vulnerabilities/sqli/?id=1%27+OR+%271%27%3D%271&Submit=Submit"
```

Expected 403 from signatures (Lab 03). Now make the block signature-independent:

### 3. Parameter validation as the block

Policy › Parameters → edit/create parameter on URL `/vulnerabilities/sqli/`:

- Name: `id`, source: any (GET/POST), **Value type: user-input**,
  **Data type: integer**, min length 1, max length 4, **regex `^[0-9]{1,4}$`**.
- Keep **Attack Signatures Check = Enabled** (defense in depth).

Apply policy. Test:

```bash
# Blocked by parameter validation (no signature needed):
curl -s -o /dev/null -w "%{http_code}\n" \
  "http://10.10.2.100/vulnerabilities/sqli/?id=1%27+OR+%271%27%3D%271&Submit=Submit"
# Still allowed:
curl -s -o /dev/null -w "%{http_code}\n" \
  "http://10.10.2.100/vulnerabilities/sqli/?id=5&Submit=Submit"
```

Check the Requests log: violation is **`Illegal parameter value`** — not
`Attack signature detected`. That's the positive model blocking.

### 4. Login parameters

Policy › Parameters → on `/login.php`:

- `username`: user-input, **alphanumeric** (DVWA allows only simple names), max 20.
- `password`: user-input, **any** data type but max length 128.
- `Login` (submit button): **static-content**, value `Login`.

Test:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -d "username=admin&password=password&Login=Login" \
  http://10.10.2.100/login.php                                    # 302/200 ok
curl -s -o /dev/null -w "%{http_code}\n" -d "username=admi%27n&password=x&Login=Login" \
  http://10.10.2.100/login.php                                    # 403 illegal parameter
```

Note: blocking login injection here **also defends the brute-force/credential-stuffing
surface** (Lab 12 builds on this).

### 5. Command-injection parameter with regex

`/vulnerabilities/exec/` POSTs `ip` (an IP address or hostname to ping):

- `ip`: user-input, data type **any**, regex **`^[a-zA-Z0-9.\-]{1,40}$`** (no spaces,
  no `;`, no `|`), max 40.

Test:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -d "ip=127.0.0.1;cat+/etc/passwd&Submit=Submit" \
  http://10.10.2.100/vulnerabilities/exec/     # 403 (space/semicolon fails regex)
curl -s -o /dev/null -w "%{http_code}\n" -d "ip=127.0.0.1&Submit=Submit" \
  http://10.10.2.100/vulnerabilities/exec/     # allowed
```

### 6. Accept parameter suggestions

Policy Building › Traffic Learning → **Parameters** suggestions (the `id`, `Submit`,
`ip` params with their observed values) → Learn → Apply. Note suggested value types/data
types come from observed traffic — review before enforcing (the suggestion is a start,
not gospel).

### 7. JSON parameters (Juice Shop)

Juice Shop (VIP on port 3000 — deploy VS `juice_vs` + policy `juice_policy` with the
same pattern as Labs 1–2, or skip to Lab 13 where it's set up properly):

- Modern APIs POST JSON bodies: `{"email":"a@b.c","password":"x"}`.
- Policy › Parameters → **JSON** parameters: name `email` (email data type), `password`
  (max length). Test a malformed-value JSON POST → `Illegal JSON parameter value`.
- Note ASM parses JSON into parameters only with a **JSON profile** on the VS
  (Lab 13 covers profiles fully; try it here if curious).

## Verification

- [ ] `id=1' OR '1'='1` blocked with **`Illegal parameter value`** (not a signature hit).
- [ ] Valid `id=5` still 200.
- [ ] Login with `'` in username blocked; normal login works.
- [ ] Command injection via `ip=127.0.0.1;cat /etc/passwd` blocked; plain IP passes.
- [ ] Parameter suggestions accepted and enforced for at least two URLs.

## Extra credit

- Set `id` to **static-content** and watch every real value fail — then fix it. This
  teaches why value type matters as much as pattern.
- Turn **Attack Signatures Check off** on the `id` parameter and re-run the sqlmap scan:
  parameter validation alone should still block the SQLi payloads — note which payloads
  get through (e.g., numeric-blind tautologies like `1 AND 1=1`? test and see).
- Add a **deny** regex on `name` (XSS page): `^[\w\s.,!?-]{0,50}$` — note this also blocks
  benign `Tom & Jerry` (unencoded `&`) and think about how you'd handle it.

## Pitfalls

- **Regex vs data type** — both apply; a `decimal` data type rejects `1,5` (comma) but
  your regex may allow it. Decide one source of truth per parameter.
- **Submit button as user-input** — attackers can send `Login=Drop+Table`; static-content
  value type locks it to the exact value.
- Parameter names are case-sensitive; DVWA uses `Submit` capital-S.

## What you learned

Per-parameter validation as the strongest ASM control: value types, data types, lengths,
regex patterns; suggestions vs manual definitions; and why parameter validation often
beats signatures for structured inputs.
