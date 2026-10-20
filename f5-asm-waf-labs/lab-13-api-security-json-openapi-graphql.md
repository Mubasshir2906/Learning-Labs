# Lab 13 — API Security: JSON Profiles, OpenAPI Policies, GraphQL

**Module 3 · Est. time 90 min · Level: advanced**

## Objectives

- Apply a **JSON profile** for body parsing + schema-ish validation.
- Build a policy **from an OpenAPI (Swagger) spec** and enforce spec-only URLs.
- Protect **GraphQL**: content profile, blocking introspection, query cost.
- Protect the Juice Shop REST API end-to-end.

## Prerequisites

Labs 0–12. Juice Shop (`10.10.1.51:3000`) and Petstore (`10.10.1.52:8080`) running.

## Concepts

- Modern APIs: JSON bodies, many endpoints, machine clients. Three ASM tools:
  - **JSON profile** — parse JSON bodies, validate well-formedness, extract parameters.
  - **OpenAPI policy** — import a swagger/OpenAPI doc → policy pre-populated with all
    URLs/parameters/methods; requests outside the spec are violations.
  - **GraphQL profile** — parse GraphQL bodies, block introspection, cost controls.

## Tasks

### 1. Petstore VS + JSON profile

Build the plumbing for the API (same pattern as Lab 01):

```
tmsh create ltm node petstore address 10.10.1.52
tmsh create ltm pool api_pool members add { petstore:8080 } monitor http
tmsh create ltm virtual api_vs destination 10.10.2.101:80 ip-protocol tcp \
  profiles add { http tcp } pool api_pool source-address-translation { type automap }
```

Create policy `api_policy` (Rapid Deployment, transparent for now — Lab 02 pattern).

**JSON profile**: Security › Application Security › Advanced Configuration ›
**Content Profiles › JSON** → Create `json_prof`:

- Enable JSON parsing; enforcement: **Malformed JSON** violation.
- Max depth 20, max structure size 100 KB (defaults; note them).

Attach JSON profile to the policy (Policy › Content profiles? or per-URL content) and
to `api_vs` (policy attach, Lab 02 pattern).

Test:

```bash
# valid
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "Content-Type: application/json" \
  -d '{"id":1,"name":"rex","status":"available"}' http://10.10.2.101/v3/pet
# malformed
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "Content-Type: application/json" \
  -d '{"id":1,,"name":"rex"}' http://10.10.2.101/v3/pet
```

**Expected:** second gets a `Malformed JSON` violation (transparent → logged; flip
blocking when confident).

### 2. OpenAPI policy from the spec

Get the spec from the service itself:

```bash
curl -s http://10.10.1.52:8080/api-docs -o petstore-openapi.json
```

Create a new policy from spec:

- Security › Application Security › Policies → Create →
  **Upload OpenAPI file** → `petstore-openapi.json` → name `petstore_api_policy`.

Inspect the generated policy: URLs list now mirrors the spec — `/v3/pet`,
`/v3/pet/{petId}`, with methods GET/POST/DELETE and typed parameters (JSON body fields).
Note statuses: spec URLs default to "allowed with staging".

Attach to `api_vs`, flip blocking, test:

```bash
# in-spec
curl -s -o /dev/null -w "%{http_code}\n" http://10.10.2.101/v3/pet/1          # 200
curl -s -o /dev/null -w "%{http_code}\n" http://10.10.2.101/v3/pet/findByStatus?status=available
# out-of-spec (not in the swagger doc)
curl -s -o /dev/null -w "%{http_code}\n" http://10.10.2.101/v3/store/inventoryX  # 403
# wrong method
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE http://10.10.2.101/v3/pet/findByStatus
```

**Expected:** out-of-spec path → `URL not allowed`; wrong method → `Method not allowed` —
the spec **is** the policy.

Re-run a spec-violating payload type:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "Content-Type: application/json" \
  -d '{"id":"<script>alert(1)</script>","name":"rex"}' http://10.10.2.101/v3/pet
```

Signature + JSON parameter checks fire on the body field.

### 3. GraphQL (Juice Shop)

Juice Shop exposes GraphQL at `/graphql`. Create `juice_vs` + `juice_policy`
(Lab 02 pattern) and flip blocking after learning.

**GraphQL profile**: Advanced Configuration › **Content Profiles › GraphQL** →
`gql_prof`: enable parsing; **block introspection** (`__schema`/`__type`);
enforce max depth 5, max cost/breadth defaults; note "Violation: GraphQL introspection".

Attach GraphQL profile to the policy (URL content profile on `/graphql`), apply, test:

```bash
# introspection — should be blocked
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}' http://10.10.2.102/graphql
# normal query — should pass
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"query":"{ AllProducts { id name } }"}' http://10.10.2.102/graphql | head -c 200
```

(v16/v17 GraphQL profile supports query depth/cost — try a deeply nested alias query and
watch `GraphQL query depth exceeded`.)

### 4. Juice Shop JSON parameters

On `juice_policy`, find login POST `/rest/user/login` (body `{"email":...,"password":...}`):

- JSON parameters: `email` (email type), `password` (max 128, masked — Lab 08).
- Test bad email format → `Illegal JSON parameter value`.

### 5. API go-live checklist (write it down)

You now have every piece; write your own ordered checklist for API onboarding:
1. learn transparent → 2. JSON profile + parameters → 3. import OpenAPI →
4. block out-of-spec → 5. signatures staged → 6. masking on sensitive fields →
7. rate limits per endpoint → 8. flip blocking → 9. monitor Requests log 24 h.

## Verification

- [ ] Malformed JSON blocked with `Malformed JSON` violation.
- [ ] OpenAPI policy: in-spec requests pass, out-of-spec path/method blocked.
- [ ] GraphQL introspection blocked; normal query passes; depth limit fires.
- [ ] Juice Shop login JSON parameters validated + password masked.

## Extra credit

- Export `petstore_api_policy` as XML and grep the URL entities — see how the spec
  mapped to policy (preview of Lab 18 lifecycle).
- Juice Shop's `/graphql` with **aliases** to amplify a query (classic GraphQL DoS) —
  verify your depth/cost settings catch it; tune and retest.
- Point sqlmap at `api_vs` and observe how much the spec-only URL model blocks
  before signatures even fire.

## Pitfalls

- **OpenAPI import ignores** non-OAS 2/3 docs and vendor extensions — clean the spec
  first (Juice Shop ships a swagger.json too, but Petstore's is cleaner for learning).
- **JSON profile without Content-Type match** — requests must come with
  `application/json` or the profile doesn't engage; attackers sometimes switch CT to
  bypass — test with `text/plain` + JSON body and set "enforce content-type" if available.
- GraphQL "block introspection" only stops `__schema` requests; attackers use cached
  schemas — depth/cost limits are the real DoS defense.

## What you learned

API-first WAF configuration: JSON parsing/validation, spec-driven policies from
OpenAPI, GraphQL hardening, and a repeatable API go-live checklist.
