# Lab 19 — Capstone: Zero-to-Hardened Juice Shop, Fully Automated

**Module 5 · Est. time 3–4 h · Level: everything**

## Objectives

- Run the complete engagement end-to-end on a new app (Juice Shop):
  deploy → learn → harden → test → block → monitor → automate.
- Prove every module's skill in one deliverable: a hardened app + policy-as-code +
  runbook.

## Prerequisites

All labs. Juice Shop at `10.10.1.51:3000` (Lab 00). Treat this as a real engagement:
**work as a team of one AppSec engineer with change control** — write your runbook as
you go.

## Scenario

Juice Shop is a modern SPA + REST/GraphQL API with a real login flow, search, basket,
file upload, and an exposed `/graphql`. You must take it to production behind BIG-IP
AWAF: no functionality broken, real attacks blocked, changes reproducible from git.

## Phases

### Phase 1 — Deliver (Labs 01–02 skills)

```
tmsh create ltm node juice address 10.10.1.51
tmsh create ltm pool juice_pool members add { juice:3000 } monitor http
tmsh create ltm virtual juice_vs destination 10.10.2.102:80 ip-protocol tcp \
  profiles add { http tcp } pool juice_pool source-address-translation { type automap }
```

Create `juice_policy` (Rapid Deployment) and attach, **transparent**.

### Phase 2 — Learn (Labs 02, 05)

Drive the app like a real user for 15–20 min:

- Register an account, log in, browse products, search, add to basket, "check out",
  open `/graphql` playground, use the REST endpoints you see in devtools
  (`/rest/products/search`, `/rest/basket/1`, ...).
- Accept URL/file-type/parameter suggestions; note JSON-heavy traffic —
  a **JSON profile** is mandatory (Lab 13).

### Phase 3 — Harden (Labs 03–09, 13)

Apply the full stack to `juice_policy`:

- [ ] Signature sets: Attack + Generic High/Medium (+ CVE), staged.
- [ ] JSON profile attached; malformed JSON blocks.
- [ ] Explicit URL model; `*` URL disallowed; methods GET/POST only.
- [ ] Parameters: login `email` (email type), `password` (masked, max len);
      API JSON parameters (product ids integer, etc.).
- [ ] Login page + session tracking + login enforcement on `/rest/*` protected APIs.
- [ ] CSRF where forms exist; **note** SPA token flows — if injection breaks the app,
      scope it (this is a real-world AWAF/SPA tension; document your decision).
- [ ] Data Guard + masking (password, tokens, email in responses/logs).
- [ ] Brute force prevention on `/rest/user/login`.
- [ ] DoS profile: modest per-IP request limit + per-URL limit on `/rest/user/login`;
      BDP enabled after baseline browsing.

### Phase 4 — Go blocking (Lab 04 workflow)

1. Enforcement readiness review — clear red flags.
2. Trusted IP for your workstation.
3. Flip blocking. **Full manual regression pass** — every user flow listed in Phase 2.
   Fix entities until clean. Record each fix in the runbook.

### Phase 5 — Attack test (Labs 03, 06, 10–12, 13)

Run the battery, record results (blocked? which violation?):

```bash
# SQLi on search
curl -s "http://10.10.2.102/rest/products/search?q=%27))%20UNION%20SELECT%20id,email,password--"
# XSS via search
curl -s "http://10.10.2.102/#/search?q=%3Cscript%3Ealert(1)%3C/script%3E"
# GraphQL introspection
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}' http://10.10.2.102/graphql
# Login brute force (10x loop)
for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code} " \
  -X POST -H "Content-Type: application/json" \
  -d "{\"email\":\"admin@juice-sh.op\",\"password\":\"wrong$i\"}" \
  http://10.10.2.102/rest/user/login; done; echo
# NoSQL injection attempt on login
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "Content-Type: application/json" \
  -d '{"email":{"$gt":""},"password":{"$gt":""}}' http://10.10.2.102/rest/user/login
# Rate-limit flood
hey -n 200 -c 10 http://10.10.2.102/
# Bot UA
curl -s -o /dev/null -w "%{http_code}\n" -A "sqlmap/1.7" http://10.10.2.102/
```

**Success bar:** all attacks blocked/mitigated; all Phase-2 user flows still work
(run them again after the attack battery).

### Phase 6 — Operate (Labs 15–16)

- Remote logging to your SIEM stand-in; alert on >10 blocked/min per IP.
- iRule: health-check bypass + `X-WAF-Reason` header to the backend.
- Custom block page with Support ID.

### Phase 7 — Automate (Labs 17–18)

- Export the finished policy as **Declarative JSON** → `waf/juice-policy.json` in a git
  repo with:
  - `waf/README.md` — your runbook (learn → harden → readiness → block → attack-test →
    monitor), with the exact commands you used.
  - `as3/` — the full AS3 declaration (VS, pool, WAF attachment).
- Apply from scratch on a throwaway policy name to prove reproducibility
  (delete `juice_policy_automation` after the test).

## Deliverables checklist

- [ ] `juice_vs` serving hardened Juice Shop; attacks blocked, users unaffected.
- [ ] Attack-test table (payload → result → violation) in the runbook.
- [ ] `juice-policy.json` (Declarative) + AS3 declaration in git, re-applied from
      scratch successfully.
- [ ] Logging flowing to the stand-in SIEM with at least one alert rule demonstrated.
- [ ] Rollback procedure documented and **tested once**.

## Extra credit

- Repeat the whole capstone against **crAPI** (docker compose, owasp/crAPI) — a harder
  multi-service API target with JWT auth. Note what changes (multiple policies per
  service? one policy with URL partitions?).
- Add an automated regression: script the Phase-5 battery (expect-403 asserts) and wire
  it into the CI/CD from Lab 17 — a WAF regression suite in ~100 lines of bash.

## Pitfalls

- **Time-box Phase 2** — Juice Shop's SPA fires many small API calls; devtools-open
  browsing captures them all. Missed calls surface in Phase 4 as 403s: that's the
  accept-and-verify loop again.
- **GraphQL + CSRF/login enforcement interplay** — `/graphql` is authenticated;
  login enforcement on it will break the playground without a session. Scope carefully.
- Don't block `curl` UAs *and* forget your own automation (the regression suite!) —
  whitelist the runner's IP like a professional.

## What you learned

Everything, once, on one app — the full WAF engineering loop you'll repeat for every
application you protect.
