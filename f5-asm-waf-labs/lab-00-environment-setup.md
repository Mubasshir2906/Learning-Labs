# Lab 00 — Build the Lab Environment

**Module 0 · Est. time 60–90 min · Level: setup**

## Objectives

- Deploy BIG-IP Virtual Edition with an ASM/AWAF-capable license.
- Deploy the three vulnerable lab apps in Docker.
- Wire the three-node topology (client ↔ BIG-IP ↔ apps).
- Set up your client attack toolkit.

## Prerequisites

- 16 GB+ free RAM, ~80 GB disk, virtualization enabled.
- Docker available on some host reachable at `10.10.1.50`.

## Tasks

### 1. Get BIG-IP VE

Pick one hypervisor; all work fine for a lab:

- **VMware Workstation/Fusion or ESXi** — download BIG-IP VE `.ova` from F5 Downloads
  (search "BIG-IP Virtual Edition", pick `All_VE*` latest v17.1). Use a **Lab license VE**
  size (1 vCPU / 2 GB works; 2 vCPU / 4 GB smoother).
- **VirtualBox** — convert the qcow2 image (`qemu-img convert -f qcow2 -O vdi`) or grab the
  community VHD.
- **Proxmox/KVM** — import the qcow2 directly.
- **Cloud** — AWS/Azure marketplace have BIG-IP VE PAYG images with ASM/AWAF; pick
  smallest size and adjust security groups to match the topology below.

Request the **30-day free trial license** at f5.com Free Trials — it includes ASM/AWAF
features. Note your **registration key** (regkey, format `XXXXX-XXXXX-...`).

### 2. Network plan

Configure two networks on the BIG-IP VM:

- **mgmt network** — NAT/host-only, BIG-IP mgmt IP `192.168.10.10/24`, gw `192.168.10.1`.
  Your workstation sits here and reaches GUI/SSH.
- **internal network** — host-only/internal, where the Docker host lives at `10.10.1.50/24`.

### 3. First-boot setup

1. Boot the VM. Console shows the setup wizard. If no wizard, use the console login
   `root` / (blank or `default` on older builds) then run `tmsh`:
   ```
   tmsh create sys management-ip 192.168.10.10/24
   tmsh create sys management-route default gateway 192.168.10.1
   tmsh create sys dns name-servers add { 192.168.10.1 }
   tmsh save sys config
   ```
2. Browse to `https://192.168.10.10`, log in `admin` / `admin` — you are forced to change
   the password. **Record it**; labs use `<admin-pass>` placeholder.
3. Run the Setup Utility: set hostname `bigip1.lab.local`, timezone, NTP.
4. License: choose **Activate** → paste regkey → activate (needs outbound internet, or use
   manual activation with the license file F5 emailed you).
5. Provisioning: System › Resource Provisioning — check that **ASM** (or **AWAF**) is
   licensed but only enable provision level **Nominal** now (do it via CLI in Lab 01).
   Leave LTM at Nominal; everything else None.

### 4. Deploy the target apps (Docker host)

On your Docker host (`10.10.1.50`, can be your workstation with port-mapped networks or a
small Linux VM on the internal network):

```bash
docker network create --subnet=10.10.1.0/24 labnet

# DVWA — deliberately vulnerable PHP app (main target, labs 1–12, 15–18)
docker run -d --name dvwa --network labnet --ip 10.10.1.50 -p 80:80 vulnerables/web-dvwa

# Juice Shop — modern Node/SPA app with REST APIs (labs 13, 19)
docker run -d --name juice --network labnet --ip 10.10.1.51 -p 3000:3000 \
  bkimminich/juice-shop

# Petstore API — clean OpenAPI 3 service (lab 13)
docker run -d --name petstore --network labnet --ip 10.10.1.52 -p 8080:8080 \
  swaggerapi/petstore3:latest
```

Verify from your workstation: `http://10.10.1.50` (DVWA login `admin`/`password`),
`http://10.10.1.51`, `http://10.10.1.52`.

> DVWA first run: log in, go to DVWA Setup, click **Create / Reset Database**, set DVWA
> Security level to **Low** (you want the app vulnerable; the WAF is the defense).

### 5. Client toolkit

On the client (`192.168.10.20`, your workstation or a Kali VM):

```bash
pip install sqlmap                      # SQL injection scanner
# hey: single-binary load generator — grab for your OS from github.com/rakyll/hey/releases
curl -I http://10.10.1.50               # sanity check reachability
```

Optional but recommended: Burp Suite Community (manual exploit crafting), Nikto, hydra.

### 6. Shell variables used by every lab

Create `lab-env.sh` (or set in PowerShell `$env:` equivalents) and source it each session:

```bash
export BIGIP=192.168.10.10
export AUTH="admin:<admin-pass>"
alias tm='ssh admin@192.168.10.10 tmsh'      # or: ssh into the box, then tmsh
```

REST smoke test:

```bash
curl -sk -u "$AUTH" https://$BIGIP/mgmt/tm/sys/version | grep -o '"Version":"[^"]*"'
```

## Verification

- [ ] `https://192.168.10.10` GUI opens; license shows ASM/AWAF as licensed.
- [ ] `http://10.10.1.50`, `:3000`, `:8080` all reachable from the client.
- [ ] REST smoke test returns the BIG-IP version JSON.
- [ ] `sqlmap --version` and `hey` run from the client.

## Pitfalls

- **License activation fails** — trial activation needs outbound 443 from the BIG-IP mgmt
  interface; proxy? use manual activation (copy challenge to F5's activation portal).
- **Docker Desktop on the same workstation** — you can still lab with it: map DVWA to
  `localhost:80` and use `10.10.2.100` VIP → need a route/self-IP adjustment; simplest is a
  dedicated Linux VM as the Docker host on the internal network.
- BIG-IP VE **stuck at boot >10 min** — give it 2 vCPU/4 GB; VE under 2 GB RAM fails to
  start sdmd (the ASM daemon needs it later).

## What you learned

How to stand up BIG-IP VE with ASM licensing, the lab's three-app target farm, and the
client toolkit you'll attack with in every subsequent lab.
