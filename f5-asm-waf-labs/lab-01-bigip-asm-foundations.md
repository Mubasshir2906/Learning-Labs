# Lab 01 — BIG-IP + ASM Architecture, First Virtual Server

**Module 1 · Est. time 60 min · Level: beginner**

## Objectives

- Provision the ASM module and understand how it sits in the traffic path.
- Build the LTM plumbing: VLAN, self-IP, node, pool, HTTP virtual server.
- Take the guided policy tour of the ASM UI.

## Prerequisites

Lab 00 complete. This lab needs **no** security policy yet — that's Lab 02.

## Concepts (read first, 10 min)

- **LTM** delivers the traffic (VIP → pool → member). **ASM** inspects it: requests are
  intercepted by an `asm` profile on the virtual server, evaluated against the attached
  **security policy**, then forwarded or blocked with a **Support ID** page.
- **Security policy** = the rulebook (allowed URLs, parameters, signatures, response
  handling). It has **learning** (suggestions from traffic) and an **enforcement mode**
  (transparent = log-only, blocking = enforce).
- ASM runs in TMM (the data plane); its decisions surface in
  **Security › Event Logs › Application › Requests**.

## Tasks

### 1. Provision ASM (Nominal)

CLI over SSH (`ssh admin@192.168.10.10`):

```
tmsh modify sys provision asm nominal
tmsh save sys config
tmsh show sys provision
```

Wait 2–5 min. TMM restarts. Verify: `tmsh show sys provision` → `asm nominal`, and the GUI
shows **Security › Application Security** menus.

> Licensing on trial keys: if `modify sys provision asm` errors with "not licensed",
> re-check the regkey covers ASM/AWAF (Lab 00).

### 2. Build the network objects

All GUI paths + tmsh equivalents:

**VLAN + self-IP**

```
tmsh create net vlan internal interfaces add { 1.2 }
tmsh create net self 10.10.1.10/24 vlan internal allow-service all
```

GUI: Network › VLANs → Create (`internal`, untagged interface 1.2); Network › Self IPs.

> VE on VMware: add a second NIC for the internal network; interface numbering `1.1`,
> `1.2`, ... For VirtualBox/KVM adjust accordingly.

**Node + pool**

```
tmsh create ltm node dvwa address 10.10.1.50
tmsh create ltm pool app_pool members add { dvwa:80 } monitor http
```

GUI: Local Traffic › Nodes/Pools.

**HTTP virtual server**

```
tmsh create ltm virtual app_vs destination 10.10.2.100:80 ip-protocol tcp \
  profiles add { http tcp } pool app_pool source-address-translation { type automap }
```

GUI: Local Traffic › Virtual Servers → Create. Name `app_vs`, Destination
`10.10.2.100:80`, Service Port 80, HTTP + TCP profiles, Source Address Translation:
Auto Map.

### 3. First requests — no WAF yet

From the client:

```bash
curl -s http://10.10.2.100/ | grep -i dvwa
```

**Expected:** DVWA's login page HTML. The VIP works; ASM is provisioned but nothing is
attached to `app_vs` yet.

Confirm the "no policy" state:

```
tmsh list ltm virtual app_vs profiles
```

### 4. The ASM UI tour (click-through)

With `app_vs` selected in mind, walk these menus — do not change anything yet:

1. **Security › Application Security › Dashboard** — attack/traffic overview (empty now).
2. **Security › Application Security › Policies** — where policies live (Lab 02).
3. **Security › Application Security › Policy Building** — learning suggestions.
4. **Security › Event Logs › Application › Requests** — the per-request verdict log.
5. **Security › Options › Application Security › Advanced Configuration** —
   low-level settings: Attack Signatures, Blocking Settings, HTTP protocol.
6. **Security › DoS Protection** — DoS/bot profile areas (Module 3).
7. **System › Configuration › Device › Licensing** — confirm AWAF features listed.

### 5. REST equivalent (reference)

```bash
source lab-env.sh
curl -sk -u "$AUTH" -H "Content-Type: application/json" -X POST \
  https://$BIGIP/mgmt/tm/ltm/virtual \
  -d '{"name":"app_vs_rest","destination":"10.10.2.101:80","ipProtocol":"tcp",
       "pool":"/Common/app_pool","profilesReference":{"items":[{"name":"http"}]}}'
```

Keep REST in mind: Lab 17 automates everything you clicked today.

## Verification

- [ ] `tmsh show sys provision` shows `asm nominal`.
- [ ] `curl http://10.10.2.100/` returns DVWA through the VIP.
- [ ] You can name: virtual server, pool, node, self-IP, and where the ASM policy
      attaches (the `asm` profile on the VS — next lab).

## Extra credit

- Add the Juice Shop pool members to `app_pool` and load-balance across DVWA + Juice Shop,
  then revert (mixed apps under one policy gets ugly — that's a lesson in itself).
- Point the BIG-IP at your DNS/NTP and verify `date` and `ntpstat` on the box — correct
  clocks matter for log forensics in Lab 15.

## Pitfalls

- **No `Security` menu after provisioning** — the GUI needs a re-login; ASM daemons take a
  few minutes on first start (`admd`, `asm` processes in `tmsh show sys proc-info`).
- **Auto Map vs no SNAT** — without `automap` (or a wildcard SNAT pool) the pool member
  sees the client IP of the internal hop and replies may route around BIG-IP; you'll see
  "no server response" timeouts.
- Monitor `http` on the pool marks members down if DVWA returns a non-200 to `GET /` after
  first-boot DB setup; fix the app first (Lab 00 step 4).

## What you learned

ASM's place in the BIG-IP traffic path, provisioning the module, and building the LTM
delivery plumbing that every later lab attaches security to.
