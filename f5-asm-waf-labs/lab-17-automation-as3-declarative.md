# Lab 17 — Automation: REST API, Declarative Policy, AS3, CI/CD

**Module 4 · Est. time 90 min · Level: advanced**

## Objectives

- Drive ASM entirely via REST (CRUD policies, attach to VS).
- Build a policy with a **Declarative Policy** JSON in one POST.
- Use **AS3** to deploy LTM + WAF as code.
- Sketch a CI/CD pipeline that applies policy changes from git.

## Prerequisites

Lab 16. Everything scripted so far has been tmsh — now it's HTTP all the way down.

## Concepts

- ASM REST: `/mgmt/tm/asm/policies` (+ subcollections: `urls`, `parameters`,
  `signature-sets`, ...). Async operations use **task objects**
  (`/mgmt/tm/asm/tasks/...`) with a `result: { state: NEW | IN_PROGRESS | COMPLETED }`.
- **Declarative Policy** = one JSON blob describing the whole policy; POST it to
  `/mgmt/tm/asm/policies` — the engine builds/updates everything. This is the
  git-friendly format.
- **AS3** = declaration posted to `/mgmt/shared/appsvcs/declare` — full application
  delivery (VS, pool, monitors) with `class: WAF_Policy` referencing an ASM policy
  or containing an inline declaration.

## Tasks

### 1. REST basics

```bash
source lab-env.sh
# list policies
curl -sk -u "$AUTH" https://$BIGIP/mgmt/tm/asm/policies \
  '?$select=name,enforcementMode,active' | python -m json.tool

# create a policy
curl -sk -u "$AUTH" -H "Content-Type: application/json" -X POST \
  https://$BIGIP/mgmt/tm/asm/policies \
  -d '{"name":"rest_policy","applicationLanguage":"utf-8",
       "templateReference":{"link":"https://localhost/mgmt/tm/asm/policy-templates/POLICY_TEMPLATE_RAPID_DEPLOYMENT"}}'

# note the returned policy id (a hash); poll:
curl -sk -u "$AUTH" https://$BIGIP/mgmt/tm/asm/policies/~Common~rest_policy \
  '?$select=id,enforcementMode'
```

### 2. Entity CRUD via REST

Add a URL and a parameter (use your policy **id** from step 1):

```bash
PID=<policy-id>
curl -sk -u "$AUTH" -H "Content-Type: application/json" -X POST \
  https://$BIGIP/mgmt/tm/asm/policies/$PID/urls \
  -d '{"name":"/vulnerabilities/sqli/","type":"explicit","performStaging":true}'

curl -sk -u "$AUTH" -H "Content-Type: application/json" -X POST \
  https://$BIGIP/mgmt/tm/asm/policies/$PID/parameters \
  -d '{"type":"explicit","name":"id","valueType":"user-input",
       "dataType":"integer","maximumLength":4,"mandatory":false}'
```

Apply the policy (async task):

```bash
curl -sk -u "$AUTH" -H "Content-Type: application/json" -X PATCH \
  https://$BIGIP/mgmt/tm/asm/policies/$PID \
  -d '{"apply":true}'    # or POST an apply-policy task; version-dependent
```

### 3. Declarative Policy

`declarative.json`:

```json
{
  "policy": {
    "name": "declarative_policy",
    "template": { "name": "POLICY_TEMPLATE_RAPID_DEPLOYMENT" },
    "applicationLanguage": "utf-8",
    "enforcementMode": "transparent",
    "blocking-settings": {
      "violations": [
        { "name": "Attack signature detected", "alarm": true, "block": true }
      ]
    },
    "filetypes": [ { "name": "php", "type": "allow" } ],
    "urls": [
      { "name": "/login.php", "type": "explicit", "method": "POST",
        "performStaging": false },
      { "name": "/vulnerabilities/sqli/", "type": "explicit" }
    ],
    "parameters": [
      { "name": "username", "type": "explicit", "valueType": "user-input",
        "dataType": "alpha-numeric" }
    ]
  }
}
```

POST it (create or update by name):

```bash
curl -sk -u "$AUTH" -H "Content-Type: application/json" -X POST \
  https://$BIGIP/mgmt/tm/asm/policies -d @declarative.json
```

Poll the task until `COMPLETED`, then verify in the GUI: URLs/parameters/filetype match
the JSON — policy-as-code achieved.

### 4. AS3 deployment

`as3.json` (tenant `Lab`, app `dvwa`):

```json
{
  "class": "AS3",
  "declaration": {
    "class": "ADC",
    "schemaVersion": "3.40.0",
    "id": "lab",
    "Lab": {
      "class": "Tenant",
      "dvwa": {
        "class": "Application",
        "template": "http",
        "serviceMain": {
          "class": "Service_HTTP",
          "virtualAddresses": ["10.10.2.103"],
          "pool": "dvwa_pool",
          "policyWAF": { "bigip": "/Common/declarative_policy" }
        },
        "dvwa_pool": {
          "class": "Pool",
          "monitors": ["http"],
          "members": [{ "servicePort": 80, "serverAddresses": ["10.10.1.50"] }]
        }
      }
    }
  }
}
```

(AS3 must be installed — GUI: iApps › Package Management, or it's preinstalled on v17.)

```bash
curl -sk -u "$AUTH" -H "Content-Type: application/json" -X POST \
  https://$BIGIP/mgmt/shared/appsvcs/declare -d @as3.json
```

Verify: new VS `Lab_dvwa_serviceMain` with the WAF policy attached; delete with
`PATCH ... {"class":"AS3","action":"remove"}` or `DELETE /mgmt/shared/appsvcs/declare/Lab`.

### 5. CI/CD sketch

`policy/declarative.json` in git; pipeline (GitHub Actions/GitLab CI) pseudo:

```yaml
stages: [deploy]
deploy-waf:
  stage: deploy
  script:
    - curl -sk -u "$AUTH" -H "Content-Type: application/json" -X POST
        https://$BIGIP/mgmt/tm/asm/policies -d @policy/declarative.json
```

Real pipelines add: policy linting (jq validation), staging BIG-IP first, task polling,
diff-vs-current (`GET policies` → compare), and rollback (re-POST previous commit).
Note F5's `ansible` collection (`bigip_asm_policy_manage`, `bigip_as3_deploy`) as the
managed alternative — one-liner playbook exercise if you use Ansible.

## Verification

- [ ] Policies listed/created/updated via REST; async tasks polled to COMPLETED.
- [ ] Declarative Policy JSON builds a policy matching its JSON.
- [ ] AS3 declaration creates VS+pool+WAF attachment; removing the tenant cleans up.
- [ ] CI/CD sketch applies a git-committed policy change.

## Extra credit

- Export an existing policy to Declarative format:
  `GET .../policies/<id>` returns a huge object; strip to the declarative subset —
  this is how teams bootstrap policy-as-code from live configs.
- Ansible: run a 10-line playbook using `f5networks.f5_modules.bigip_asm_policy_manage`
  to import/apply `declarative.json`.

## Pitfalls

- **Tasks, not fire-and-forget** — every apply/create returns a task link; poll it or
  you'll race your own changes.
- **`~Common~` tilde encoding** in URLs (`/policies/~Common~name`) — in curl,
  quote the whole URL; `~` is fine, spaces aren't.
- AS3 `WAF_Policy` with inline `declaration` requires the AS3 schema version that
  supports it — pin `schemaVersion` and read the AS3 release notes.

## What you learned

Full-code management of ASM: raw REST + async tasks, Declarative Policy as the canonical
policy format, AS3 for the delivery stack, and the CI/CD pattern to make WAF changes
reviewable and repeatable.
