# Lab 09 — IP/Geo Controls & HTTP Protocol Hardening

**Module 2 · Est. time 60 min · Level: intermediate**

## Objects

- Blacklist IPs; enforce geolocation deny.
- Harden HTTP protocol settings: header/URL/cookie length limits, chunked handling,
  malformed request handling.
- Understand what the `Malformed HTTP...` violations catch.

## Prerequisites

Lab 08. `dvwa_policy` blocking.

## Concepts

- ASM's **IP Address** policy section: whitelist (trusted — bypass), blacklist (always
  block), and **geolocation** enforcement (deny/allow by country; needs a geo-licensed
  BIG-IP — trial VEs include it).
- **HTTP protocol controls** define limits and strictness: request/URL/header/cookie
  lengths, max headers, chunked body limits, "malformed" checks (bad request lines,
  spaces, encoding). These catch exploitation frameworks that send weird requests before
  any signature matches.

## Tasks

### 1. Blacklist an IP

Policy › **IP Address › Blacklist** → add a fake attacker IP `203.0.113.66`.
To test from a real different source, run a request from the Docker host instead
(or use Burp's "Upstream proxy" trick); simplest: temporarily add **your own** client IP,
test, remove:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://10.10.2.100/login.php
```

**Expected:** 403 with `IP is blacklisted` in the Requests log. Remove your IP after.

### 2. Geolocation

1. Policy › **IP Address › Geolocation Enforcement**.
2. Deny a country (pick one you can simulate — or use the GUI's "simulate" approach:
   deny a random country, then spoof with an iRule? No — instead, deny list includes
   countries of your Docker host's public egress if the lab has internet; observe via
   **lookups**: Requests log shows geo for each client IP).
3. Practical check: view any request's geolocation data in the Requests log details —
   confirm lookups resolve (even if to "Reserved" for private IPs).

> Without an external source, geo-enforcement can't be truly tested in a private lab.
> Document the config, verify lookup tables load
> (`tmsh show sys geoip`? — check version), and move on.

### 3. HTTP protocol limits

Policy Settings › **HTTP Protocol Compliance** (or Advanced Configuration on older):

- Max request headers: 20. Max header length: 8192. URL length: 2048.
  Cookie length: 4096. Max body (POST) size: 5 MB? Check current values first with
  `tmsh list asm policy dvwa_policy protocol-settings` — record defaults, then tighten.

Tests:

```bash
# URL too long
python -c "print('A'*2500)" > /tmp/l
curl -s -o /dev/null -w "%{http_code}\n" "http://10.10.2.100/login.php?p=$(cat /tmp/l)"
# Too many headers
curl -s -o /dev/null -w "%{http_code}\n" -H "X-1: a" -H "X-2: a" ... (repeat to 25) \
  http://10.10.2.100/login.php
```

**Expected:** `Illegal request length` / `Too many request headers` violations, 403.

### 4. Malformed request handling

Ensure "Malformed HTTP request" checks are enforced (default in blocking) and test:

```bash
# raw malformed request via netcat/openssl or python socket
printf 'GET /login.php HTTP/1.1\r\nBadHeader no-colon\r\nHost: x\r\n\r\n' | nc 10.10.2.100 80
# double-encoding / control chars
curl -s -o /dev/null -w "%{http_code}\n" "http://10.10.2.100/login.php%252e%252e"
```

Observe `Malformed HTTP URL` / `Malformed HTTP header` violations. Scanners like Nikto
emit these constantly — protocol checks alone kill much scanner noise.

### 5. Run Nikto to see protocol + signature synergy

```bash
# nikto via docker to avoid host install:
docker run --rm --network host sullo/nikto -h http://10.10.2.100 2>&1 | tail -n 30
```

In the Requests log, group by violation: note the mix of
`Malformed HTTP URL`, `Attack signature detected`, `URL not allowed` —
a full-stack demonstration of why protocol hardening matters.

### 6. Meta characters note

Older ASM had "meta character" enforcement (`'`, `"`, `;`, `<`, `>`, `&`). In v15+ these
were replaced by **evasion signatures**. If your policy shows them (older versions),
explore; on v17 you'll instead find them folded into signature detection. Know both —
interviews ask.

## Verification

- [ ] Blacklisted IP → 403 `IP is blacklisted`.
- [ ] Over-long URL / excess headers → 403 with length violations.
- [ ] Malformed requests → 403 with `Malformed HTTP...` violations.
- [ ] Nikto run produces a violation histogram with at least 3 distinct violation types.

## Extra credit

- Set **"enforce RFC 2616/7230"** style toggles (check request line, header name rules)
  and craft requests that violate each specific rule (CRLF injection attempts).
- Compare CPU (`tmsh show sys performance`) during the Nikto run with limits
  on vs limits off — protocol checks are cheap; signatures are not.

## Pitfalls

- **Tightening cookie/header limits** breaks legit apps with big auth headers
  (OIDC tokens in cookies) — measure real traffic percentiles in learning before tightening.
- Geo-licensing: trial VEs include geo DB, but updates need the license to allow it;
  check **System › License** for "Geolocation" feature.
- Blacklist is per-policy, not global — a global version exists via
  **Security › Options › Application Security › IP Intelligence** (a different feature —
  IP Intelligence, needs external feed; note the distinction).

## What you learned

Network-layer blocks (blacklist/geo) and transport-shape defenses (protocol limits,
malformed handling) that stop scanners before signatures ever fire.
