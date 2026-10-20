# Lab 18 — Policy Lifecycle: Export/Import, Diff, Cloning, Upgrades, Tuning

**Module 4 · Est. time 75 min · Level: intermediate**

## Objectives

- Export/import policies (XML) and clone policies per environment.
- Diff two policies and understand what changed.
- Manage signature updates and policy version bumps across BIG-IP upgrades.
- Apply performance tuning basics.

## Prerequisites

Lab 17. Several policies exist (`dvwa_policy`, `petstore_api_policy`, ...).

## Concepts

- Policy portability: export to **XML** (or Declarative JSON) → import into another
  BIG-IP (prod→staging→prod pattern), with signature-version compatibility rules.
- **Policy diff** shows entity-level differences (useful for audit: what did that
  change-control actually change?).
- Cloning: `dvwa_policy` → `dvwa_policy_v2` — test changes on the clone, swap VS
  attachment atomically.

## Tasks

### 1. Export/import

GUI: Policies › dvwa_policy › Export (XML file). CLI:

```
tmsh save asm policy dvwa_policy to-file /var/tmp/dvwa_policy.xml
tmsh load asm policy /var/tmp/dvwa_policy.xml name dvwa_policy_copy
tmsh list asm policy one-line | grep -o 'name [a-z_]*'
```

Import into a second BIG-IP (if you have one; otherwise the copy) and note the
**signature version check**: exported policy records the signature version; importing
into a box with a newer version prompts for a "learn"/"enforce" decision on new
signatures.

### 2. Policy diff

GUI: Policies › **Tools › Compare** (Import policy file or select two policies) →
review: URLs added/removed, parameter changes, signature-set changes.

tmsh/REST alternative: export both to XML and diff the files:

```bash
diff dvwa_policy_v1.xml dvwa_policy_v2.xml | grep -E '^[<>]' | head -n 40
```

### 3. Clone & atomic swap

```
tmsh create asm policy dvwa_policy_v2 clone-name dvwa_policy? # or copy via export/import
```

Workflow:

1. Make changes on `dvwa_policy_v2` (add an entity, enforce a signature set).
2. Test against the clone by attaching it to a scratch VS (or temporarily to `app_vs`
   during a maintenance window).
3. Swap production: `tmsh modify ltm virtual app_vs profiles modify { asm { policy dvwa_policy_v2 } }`
4. Keep `dvwa_policy` as rollback — swap back is one command.

### 4. Signature updates

- Security › Options › Application Security › **Attack Signatures › Update check**
  (needs internet or the signature file from F5 downloads).
- Schedule auto-updates: System › Scheduled Tasks? (v17: Security › Options ›
  Attack Signatures → configure update frequency).
- After an update, **new signatures enter staging** by default — your enforced set is
  stable; staged ones log. Verify: compare staged-signatures count before/after.

### 5. BIG-IP upgrade considerations (read + note)

- Policies survive upgrades; ASM migrates policy schema (export a backup first).
- The signature store version pins during upgrade — check
  `Security › Options › Policy Building › ... version notes` after upgrade.
- Rule of thumb for runbooks: upgrade UAT → run regression attacks (your lab suite!)
  → compare block behavior → promote.

### 6. Performance tuning basics

- **CPU profile**? (v16+: System › Resource Provisioning → CPU Profiles) — for L7-heavy
  boxes choose balanced vs performance; lab box is fine on default.
- Policy-side costs, most→least: attack signatures (esp. generic sets) > JSON/GraphQL
  parsing > Data Guard > URL matching. Measure: `tmsh show sys performance`
  during your sqlmap run; try disabling the Medium-accuracy set and compare CPU.
- Learning overhead: disable traffic learning once policies are mature
  (Policy › Learning mode → Disabled) — suggestions cost CPU.
- Logging: violations-only in steady state (Lab 15).

## Verification

- [ ] Exported/imported/cloned a policy; clone attached and swapped atomically.
- [ ] Diff shows your intentional change and nothing else.
- [ ] Signature update ran; new signatures staged (count verified).
- [ ] CPU comparison recorded with/without Medium generic set.

## Extra credit

- Build a **policy versioning convention** (names + git-tracked Declarative JSON) and
  write your team's release runbook: change → clone → test → swap → observe 24 h → tag.
- Practice rollback: apply a breaking change (disallow `/login.php`!), confirm the
  outage in the Requests log, then roll back in under 2 minutes.

## Pitfalls

- **Import signature-version mismatch** — old XML into a much-newer box asks to
  "learn" new signatures; decide per environment (staging: learn; prod: review first).
- **Clone sprawl** — `_v2`, `_v2_final`, `_test3` rot; automate naming + deletion in
  the runbook.
- Learning-disabled policies still log **suggestions?** no — suggestions stop; but
  staging timers keep running; review staged items after every signature update.

## What you learned

The operational lifecycle: export/import/clone, atomic VS swaps for zero-downtime policy
changes, diffing for audit, signature-update discipline across upgrades, and where the
CPU actually goes.
