# Lab 16 — iRules ↔ ASM Integration

**Module 4 · Est. time 60 min · Level: advanced**

## Objectives

- Use ASM iRule events: `ASM_REQUEST_BLOCKING`, `ASM_REQUEST_DONE`,
  `ASM_RESPONSE_BLOCKING`, `ASM_RESPONSE_DONE`.
- Selectively **disable/enable** ASM with `ASM::disable` / `ASM::enable`.
- Add headers with violation context (`ASM::reason`, `ASM::support`, `ASM::status`).
- Build one practical rule: health-check bypass + backend SIEM header.

## Prerequisites

Lab 15. Blocking policy attached to `app_vs`.

## Concepts

- ASM evaluates each request; **iRule events let your iRule run before/after the ASM
  decision**. Events (in order):
  `ASM_REQUEST_BLOCKING` (violation detected, before decision), `ASM_REQUEST_DONE`
  (after decision), then on the response `ASM_RESPONSE_BLOCKING` / `ASM_RESPONSE_DONE`.
- **`ASM::disable`** must run in `ASM_REQUEST_DONE` context (after ASM classified) for
  the *current* request, or use `HTTP::path`-based decision in `ASM_REQUEST_BLOCKING`.
- Key commands: `ASM::status` (violations string), `ASM::reason` (why blocked),
  `ASM::support` (support ID), `ASM::payload <size> <offset>` (inspect body),
  `ASM::policy <name>` (per-request policy override), `ASM::enable/disable`.

## Tasks

### 1. Enable iRule events on the policy

Policy › **Advanced Settings / iRule events**: enable
"iRule events" (both request and response events must be enabled for most commands).

### 2. Health-check bypass

Create an iRule (Local Traffic › iRules):

```tcl
when ASM_REQUEST_DONE {
    if { [HTTP::path] starts_with "/health" } {
        ASM::disable
    }
}
```

Attach to `app_vs` (GUI) or:

```
tmsh create ltm rule asm_health_rule when ASM_REQUEST_DONE { if { [HTTP::path] starts_with "/health" } { ASM::disable } }
tmsh modify ltm virtual app_vs rules add { asm_health_rule }
```

Test: `curl -s -o /dev/null -w "%{http_code}\n" http://10.10.2.100/health`
(create `/health` on DVWA first: `docker exec dvwa sh -c 'echo ok > /var/www/html/health'`).

**Expected:** 200 (bypassed), even though the URL may not be in the policy.

### 3. Log blocks with context

```tcl
when ASM_REQUEST_BLOCKING {
    log local0. "ASM_BLOCK: [IP::client_addr] [HTTP::method] [HTTP::uri] \
      violations=[ASM::status] support=[ASM::support]"
}
```

Fire an attack, then read:

```bash
curl -s -o /dev/null "http://10.10.2.100/vulnerabilities/sqli/?id=1%27+OR+%271%27%3D%271&Submit=Submit"
ssh admin@192.168.10.10 "tail -n 5 /var/log/ltm | grep ASM_BLOCK"
```

### 4. Pass violation context to the backend

```tcl
when ASM_REQUEST_DONE {
    if { [ASM::status] contains "VIOLATION" } {
        HTTP::header insert X-WAF-Flag "1"
        HTTP::header insert X-WAF-Reason [ASM::reason]
    }
}
```

Run in transparent mode (policy Settings → transparent) and watch the backend receive
the header (DVWA won't show it; use `nc -l -p 9999` as a dummy pool member or check with
a request-echo service).

### 5. Selective enable (reverse pattern)

Policy detached/ASM disabled on the VS; enable only for sensitive paths:

```tcl
when HTTP_REQUEST {
    if { [HTTP::path] starts_with "/vulnerabilities" } {
        ASM::enable /Common/dvwa_policy
    }
}
```

(`ASM::enable <policy>` — per-request policy selection: multi-policy VS patterns.
If your version doesn't support policy-name syntax, note the limitation.)

### 6. Body inspection

```tcl
when ASM_REQUEST_BLOCKING {
    if { [HTTP::header Content-Length] > 0 && [HTTP::header Content-Length] < 4096 } {
        ASM::payload collect 0 [HTTP::header Content-Length]
        log local0. "body=[ASM::payload 0 [HTTP::header Content-Length]]"
    }
}
```

POST an attack and see the raw body in the log — useful for custom SIEM enrichment.

## Verification

- [ ] `/health` bypasses ASM (200) while attacks on other paths block.
- [ ] `ASM_BLOCK` log lines appear with violations + support ID.
- [ ] Backend receives `X-WAF-Reason` on violation requests (transparent mode).
- [ ] Body logging works on a POSTed payload.

## Extra credit

- **Block-page override**: in `ASM_RESPONSE_BLOCKING`, replace the response body with
  your own JSON API error (check `ASM::payload replace` availability) — the pattern for
  consistent API error formats.
- Chaos drill: disable the policy on the VS and verify the iRule-only `ASM::enable`
  pattern still protects `/vulnerabilities/*`.

## Pitfalls

- **iRule events disabled** → your iRule never fires; always enable in policy first.
- **`ASM::disable` in `HTTP_REQUEST`** doesn't work — it must run in an ASM event
  (or set a flag in `HTTP_REQUEST` and act in `ASM_REQUEST_DONE`).
- Order of operations: ASM runs before LTM HTTP class/iRule request events for policy
  decisions; response events run after. When in doubt, log and observe.

## What you learned

ASM's iRule event surface: selective bypass/enable, violation context (`ASM::status`,
`reason`, `support`), body inspection, and the practical health-check + header-enrichment
patterns used in real deployments.
