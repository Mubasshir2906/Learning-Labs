# Lab 11 — Behavioral DoS (BDP) & Bot Defense

**Module 3 · Est. time 90 min · Level: advanced · AWAF**

## Objectives

- Enable **Behavioral & Stress-based Detection (BDP)** and let it learn a baseline.
- Simulate a behavioral anomaly and watch BDP mitigate adaptively.
- Configure **Bot Defense**: bot signatures + mitigation actions.
- Understand device-ID/CAPTCHA options and the F5 anti-bot cloud dependency.

## Prerequisites

Lab 10 (DoS profile exists; keep static limits off or high so BDP is the star).

## Concepts

- **BDP** profiles traffic per site (and optionally per URL) — TPS, client IPs, URL
  distributions, user-agent mix. Deviations from the learned baseline trigger
  **mitigation escalation**: rate-limit → block → (heavy) block-all, then relax.
- BDP needs **learning time** (default ~1 day of traffic; lab-shorten it) and works best
  with stable traffic patterns.
- **Bot Defense (AWAF)**: classify automated clients by signature (User-Agent),
  optionally require a challenge; **F5-managed anti-bot service** (cloud) adds JS
  challenges/device ID — needs outbound connectivity + AWAF licensing; the lab uses the
  local signature-based mode.

## Tasks

### 1. Reset the DoS profile for BDP

Edit `dvwa_dos_profile`:

- Disable (or raise) the static request limits from Lab 10.
- Enable **Behavioral & Stress-based Detection** → Mitigation: **None (transparent)**
  first.
- Heavy URL protection: enable with defaults; add `/login.php` as a URL to protect.

Learning period shortcut:

```
tmsh modify security dos profile dvwa_dos_profile application dos-configuration? \
```

(v17 exposes BDP learning time in the GUI: Application → BDP → "Minimum learning
period" — set 5 minutes for lab.)

### 2. Build a normal baseline (10+ min)

Run mixed traffic resembling humans:

```bash
# in one terminal: slow browser-like browsing
for i in $(seq 1 60); do curl -s -o /dev/null http://10.10.2.100/login.php; sleep 2; done
# in another: occasional form posts
for i in $(seq 1 10); do curl -s -o /dev/null -d "ip=127.0.0.1&Submit=Submit" \
  http://10.10.2.100/vulnerabilities/exec/; sleep 5; done
```

Check **DoS Dashboard → BDP** — "Learning" state, baseline building.

### 3. Behavioral attack

From a machine/IP different from the baseline traffic if possible (or same, since BDP
also reacts to TPS/URL spikes):

```bash
hey -n 1000 -c 20 -z 60s http://10.10.2.100/vulnerabilities/fi/?page=../../etc/passwd
```

Watch the DoS Dashboard: attack detected (`DoS attack: Behavioral`), with the
signature of the attack (URL concentration, TPS spike). With mitigation **None**,
note "attack detected, no mitigation" — this is the tuning/validation phase.

### 4. Turn on mitigation

Edit profile → BDP mitigation: **Rate limit → Block** escalation (keep defaults).

Repeat the behavioral flood.

**Expected:** Dashboard shows mitigated client IPs; `hey` histogram shows non-2xx
ramping; normal browsing from your IP still works (BDP targets the anomalous pattern,
not everyone).

### 5. Bot Defense — signature mode

Security › **DoS Protection › Bot Defense Profiles**? (v17: DoS Profile → Bot Defense
section, or separate **Bot Defense Profile** object — locate in your version.)

- Enable **Bot signature detection**.
- Classify: known search-engine UAs (Googlebot, bingbot) → **allow**.
  Custom: any request with `User-Agent: curl/*` or `python-requests/*` → **bad bot
  → block**.
- Note the "Rate-based bot detection" option (TPS per IP) — leave off here.

Test:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -A "curl/8.0" http://10.10.2.100/login.php
curl -s -o /dev/null -w "%{http_code}\n" -A "Mozilla/5.0 (X11; Linux) Firefox/120" \
  http://10.10.2.100/login.php
```

**Expected:** curl UA → blocked (`Bot Defense` / `Client contains signature of malicious
client`), Firefox UA → allowed. Check the **Bot Defense log**
(Security › Event Logs › Traffic › Bot Security or the DoS dashboard).

### 6. Device ID / CAPTCHA (understand, optional)

If your VE license includes the **F5 anti-bot cloud service** and has outbound access:

- Enable CAPTCHA mitigation for bad bots on the login page and observe the challenge.
- If not licensed/offline: read the docs' device-ID flow (JS collector → `f5vmfp` cookie
  → reputation service) and note it as the production pattern for credential-stuffing
  farms (Lab 12 revisits).

## Verification

- [ ] BDP learned a baseline (Dashboard shows learning → active).
- [ ] Behavioral flood detected with and without mitigation; normal users unaffected
      during mitigation.
- [ ] curl-UA blocked as bot; browser UA allowed; log entries visible.
- [ ] Static limits vs BDP distinction documented in your notes (when to use which).

## Extra credit

- Attack with a *distributed* pattern if you have 2+ machines: BDP's per-IP limits don't
  trip, but the aggregate baseline does — see the difference in the dashboard.
- Write down the escalation ladder from the BDP docs (rate limit → block →
  heavy/block-all → clear) and map it to what you observed.

## Pitfalls

- **BDP needs real traffic** — attacking before baseline builds yields nothing; always
  run the learning phase.
- **Lab-shortened learning** makes baselines brittle; expect a false mitigation or two.
  Tune `heavy`/`light` thresholds down/up and observe.
- **Bot Defense by UA alone is spoofable** — attackers send `Mozilla/5.0`; device-ID /
  anti-bot service is the durable answer (licensing + latency trade-offs).

## What you learned

Behavioral DoS: baseline learning, anomaly detection, adaptive escalation; bot
classification with signatures; and where the cloud-backed anti-bot service fits.
