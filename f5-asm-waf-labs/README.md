# F5 ASM / Advanced WAF — Complete Hands-On Lab Series

A progressive, lab-first course that takes you from zero to fully operating, tuning, and
automating F5 Application Security Manager (ASM) / Advanced WAF (AWAF) on BIG-IP.

**Target version:** BIG-IP v17.1.x (works on v16.1+ with minor UI differences).
**Format:** every lab is a do-it session — GUI steps + `tmsh` + REST API + attacks you fire
at your own lab app and watch the WAF catch.

---

## Who this is for

- Security engineers / WAF admins who have never touched BIG-IP, **or**
- Network engineers who know LTM but not ASM, **or**
- AppSec folks preparing for F5's WAF-focused roles/certifications.

You should know basic HTTP (methods, headers, cookies, status codes) and basic TCP/IP.
Scripting with `curl` helps; we teach everything BIG-IP-specific from scratch.

---

## Lab topology (used in every lab)

```
  [ Client / Attacker ]          [ BIG-IP VE ]                 [ Web Apps (Docker) ]
  Kali or any OS                 mgmt:  192.168.10.10          10.10.1.50 :  80  DVWA (PHP)
  192.168.10.20  ───────────────►  VIP:  10.10.2.100  ───────►  10.10.1.51 :3000  Juice Shop (Node)
  curl, sqlmap, Burp,            self-IP internal:             10.10.1.52 :8080  Petstore API
  hey/hping3, hydra              10.10.1.10/24
```

- **Client** — your workstation or a Kali VM. Runs browsers, curl, and attack tools.
- **BIG-IP VE** — F5 BIG-IP Virtual Edition with an ASM/AWAF license (see Lab 00).
- **Targets** — deliberately vulnerable apps in Docker, deployed in Lab 00.

## Naming conventions (used in all labs)

| Object | Name |
|---|---|
| Virtual server | `app_vs` (VIP `10.10.2.100:80`) |
| Pool | `app_pool` (members `10.10.1.50:80`) |
| ASM policy | `dvwa_policy` (and friends) |
| Partition | `/Common` everywhere |

## Course map

| Module | Lab | Topic | Time |
|---|---|---|---|
| **0 — Setup** | 00 | Build the lab: BIG-IP VE, Docker apps, client tools | 60–90 min |
| **1 — Fundamentals** | 01 | BIG-IP + ASM architecture, first virtual server, policy tour | 60 min |
| | 02 | Your first security policy: learning mode, violations, suggestions | 75 min |
| | 03 | Attack signatures: sets, accuracy, staging | 75 min |
| | 04 | Going blocking + false-positive triage workflow | 75 min |
| **2 — Policy deep dive (positive security)** | 05 | Entities: URLs, file types, methods, headers, cookies | 75 min |
| | 06 | Parameters & positive security validation | 90 min |
| | 07 | Sessions: login pages, login enforcement, session tracking, CSRF | 90 min |
| | 08 | Data protection: Data Guard, masking, custom blocking pages | 60 min |
| | 09 | IP/geo controls & HTTP protocol hardening | 60 min |
| **3 — Threats** | 10 | L7 DoS protection & rate limiting | 75 min |
| | 11 | Behavioral DoS (BDP) & Bot Defense | 90 min |
| | 12 | Brute force & credential stuffing defense | 60 min |
| | 13 | API security: JSON, OpenAPI policies, GraphQL | 90 min |
| | 14 | Vulnerability assessment & suggested mitigations | 60 min |
| **4 — Operations** | 15 | Logging, remote logging/SIEM, event forensics | 75 min |
| | 16 | iRules ↔ ASM integration | 60 min |
| | 17 | Automation: tmsh, REST, Declarative Policy, AS3, CI/CD | 90 min |
| | 18 | Policy lifecycle: export/import, diff, cloning, upgrades, tuning | 75 min |
| **Graduation** | 19 | Capstone: zero-to-hardened Juice Shop, fully automated | 3–4 h |

## How to run the labs

1. Do them **in order** — each lab builds on the previous state.
2. Every lab has: objectives → prerequisites → tasks (GUI + `tmsh` + REST) →
   **verification tests** → extra credit → pitfalls → what you learned.
3. Keep a scratch notes file per lab; you'll reuse configs in the capstone.

## Prerequisites checklist (all built in Lab 00)

- Hypervisor able to run BIG-IP VE: VMware Workstation/Fusion, VirtualBox, Proxmox,
  KVM, or ESXi (cloud notes included for AWS/Azure).
- BIG-IP VE image + license: F5's free 30-day trial license includes ASM/AWAF features.
  Download from F5 Downloads (BIG-IP Virtual Edition) and request a trial key at
  f5.com → Free Trials.
- Docker (Docker Desktop on Windows/macOS, or Docker Engine on a Linux VM) for target apps.
- Client tools: curl, Python 3, `sqlmap` (`pip install sqlmap`), `hey` (load gen,
  github.com/rakyll/hey), optional Burp Suite Community, optional Kali VM for
  sqlmap/nikto/hydra out of the box.

## Version notes

- F5 renamed ASM → **Advanced WAF (AWAF)** around v15. The engine is the same module
  (`asm`); AWAF adds Behavioral DoS, Bot Defense, Login Enforcement, Credential Stuffing
  Defense, OpenAPI policies. Labs 10–13 use AWAF features and are marked accordingly.
- UI paths are given for v17.1. On older versions some pages moved; search by menu name.
- All REST examples hit `https://192.168.10.10/mgmt/tm/...` with
  `-u admin:<password> -k`. Set `BIGIP=192.168.10.10` and `AUTH="admin:<password>"`
  shell variables once (Lab 01) to copy/paste the commands as-is.
