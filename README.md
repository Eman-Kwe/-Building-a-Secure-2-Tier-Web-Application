# Secure 2-Tier Web Application on Azure
 
**Status:** Not yet built. Steps, screenshots, and results get filled in after deployment.
 
## 🎬 Video Walkthrough
 
<!-- After recording: replace YOUR-VIDEO-ID with your Loom video ID (the last part of the share link) -->
[![Loom](https://img.shields.io/badge/Loom-Watch%20Walkthrough-8B5CF6)](https://www.loom.com/share/YOUR-VIDEO-ID)
 
---
 
## Project Overview
 
Builds a two-tier network in Azure: a web server with a public IP in one subnet, and a database server with no public IP in another. Network security rules make sure only the web tier can reach the database.
 
The common shortcut is to put every server on a public IP and rely on passwords. That works until one weak credential or one open port exposes the data. The standard design keeps the database off the internet entirely and controls the path to it. This lab builds that design and then proves it works by testing what is allowed and what is blocked.
 
## Skills Demonstrated
 
- Azure Virtual Network design: address space, subnets, public vs. private tiers
- Network Security Groups: rule priority, allow/deny logic, service tags
- Deploying Linux VMs with SSH key authentication
- Reaching a private server through a jump host, using agent forwarding instead of copying keys
- Validating security controls by testing what should fail as well as what should work
- Resource group organization and naming conventions
## Architecture
 
![Architecture diagram](docs/architecture.svg)
 
- The internet can reach only `vm-web-01` (HTTP 80, and SSH 22 from my IP).
- `vm-db-01` has no public IP. It accepts SSH and database traffic only from the web subnet `10.0.1.0/24`.
- Editable source: [`docs/architecture.drawio`](docs/architecture.drawio)
## Prerequisites
 
- [ ] Active Azure subscription (Free Tier works)
- [ ] Terminal with an SSH client
- [ ] Basic comfort with Linux commands and SSH keys
## Naming Conventions
 
| Resource | Name |
|---|---|
| Resource group | `rg-lab02-manuel` |
| VNet | `vnet-lab02` (`10.0.0.0/16`) |
| Public subnet | `snet-web` (`10.0.1.0/24`) |
| Private subnet | `snet-db` (`10.0.2.0/24`) |
| Web VM | `vm-web-01` |
| Database VM | `vm-db-01` |
| SSH key | `key-lab02` |
 
Region for everything: East US.
 
## Project Steps
 
Each step ends with a screenshot to capture. Save it in `docs/screenshots/` with the filename shown, then replace the instruction block with the embedded image. See the [screenshot guide](#screenshot-guide) for what to hide.
 
### Part 1: Build the network
 
1. In the Azure portal, search for **Virtual networks** and click **Create**.
2. Create a new resource group named `rg-lab02-manuel`.
3. Name the VNet `vnet-lab02`, region East US.
4. On the **IP addresses** tab, set the address space to `10.0.0.0/16`. Replace the default subnet with `snet-web` (`10.0.1.0/24`) and `snet-db` (`10.0.2.0/24`).
5. Click **Review + create**, then **Create**.
> **Screenshot 01: `01-vnet-subnets.png`**
> `vnet-lab02` → **Subnets**, showing both subnets and their address ranges.
 
### Part 2: Deploy the web server
 
1. Search for **Virtual machines** → **Create** → **Azure virtual machine**.
2. Resource group `rg-lab02-manuel`, name `vm-web-01`, region East US.
3. Image: **Ubuntu Server 22.04 LTS**. Size: **Standard_B1s** (pick another small size if it is unavailable in your region).
4. Authentication: SSH public key, username `azureuser`, new key pair named `key-lab02`.
5. Public inbound ports: **Allow selected ports** → **HTTP (80)** and **SSH (22)**.
6. On the **Networking** tab, confirm the VNet is `vnet-lab02` and the subnet is `snet-web`. Public IP: create new, Standard.
7. Click **Review + create** → **Create**. Download `key-lab02.pem` when prompted and keep it out of any Git folder.
8. Restrict SSH: open `vm-web-01` → **Networking** → the SSH inbound rule. Change **Source** from `Any` to **IP Addresses** and enter your own public IP.
9. Optional: SSH in, run `sudo apt update && sudo apt install -y nginx`, then open `http://<web-public-ip>` in a browser to confirm the web tier is reachable.
> **Screenshot 02: `02-web-vm-overview.png`**
> `vm-web-01` **Overview**, showing the public IP and `vnet-lab02/snet-web`. Blur the public IP.
>
> **Screenshot 03: `03-web-nsg-ssh-restricted.png`**
> The web VM's NSG inbound rules, with SSH limited to one source IP and HTTP open. Blur your IP.
>
> **Screenshot 04: `04-nginx-default-page.png`** (optional)
> Browser showing the default nginx page, with the IP in the address bar blurred.
 
### Part 3: Deploy the database server
 
1. Create a second VM named `vm-db-01`, same resource group, region, image, and size.
2. Authentication: use existing key `key-lab02`.
3. Public inbound ports: **None**. This server never takes traffic from the internet, so there is nothing to open.
4. On the **Networking** tab, set the VNet to `vnet-lab02` and the subnet to `snet-db`.
5. Set **Public IP** to **None**. This is the step that keeps the server off the internet.
6. Click **Review + create**, then **Create**.
7. Open `vm-db-01` and note the **Private IP address**. It should be `10.0.2.4`.
> **Screenshot 05: `05-db-vm-no-public-ip.png`**
> `vm-db-01` **Overview**, showing private IP `10.0.2.4`, subnet `snet-db`, and no public IP.
 
### Part 4: Test connectivity between tiers
 
Azure's default rules allow all traffic inside a VNet, so this should work before any custom rules exist.
 
```bash
chmod 400 key-lab02.pem
ssh-add key-lab02.pem
ssh -A azureuser@<web-public-ip>
```
 
Then, from `vm-web-01`:
 
```bash
ping -c 4 10.0.2.4
ssh azureuser@10.0.2.4
hostname
```
 
`-A` forwards the key through the web server, so the private key is never copied onto it. One-command alternative:
 
```bash
ssh -i key-lab02.pem -J azureuser@<web-public-ip> azureuser@10.0.2.4
```
 
> **Screenshot 06: `06-ping-web-to-db.png`**
> Terminal on `vm-web-01` (prompt shows `azureuser@vm-web-01`) with four ping replies from `10.0.2.4`.
>
> **Screenshot 07: `07-ssh-jump-to-db.png`**
> Terminal after the jump, with the prompt showing `azureuser@vm-db-01` and the output of `hostname`.
 
### Part 5: Lock down the database server
 
Right now any subnet in the VNet could reach the database. Add three rules to `vm-db-01`'s NSG: **Networking** → click the NSG → **Inbound security rules** → **Add**.
 
| Priority | Name | Source | Service / Port | Action |
|---|---|---|---|---|
| 100 | `Allow-Web-SSH` | IP addresses `10.0.1.0/24` | SSH, 22 | Allow |
| 110 | `Allow-Web-DB` | IP addresses `10.0.1.0/24` | Custom, 3306 or 5432 (only if you install a database) | Allow |
| 4000 | `Deny-VNet-Other` | Service tag `VirtualNetwork` | Custom, `*` | Deny |
 
The deny rule at 4000 is checked before Azure's built-in "allow VNet" rule at 65000, so anything not explicitly allowed gets dropped.
 
Then test:
 
1. From `vm-web-01`, ping `10.0.2.4`. It should now fail, because ICMP is not on the allow list.
2. From `vm-web-01`, SSH to `10.0.2.4`. It should still work.
3. From your own computer, run `ssh -i key-lab02.pem azureuser@10.0.2.4`. It should time out.
> **Screenshot 08: `08-db-nsg-rules.png`**
> The `vm-db-01` NSG inbound rules showing priorities 100, 110, and 4000. Bonus: capture **Effective security rules** on the VM's network interface.
>
> **Screenshot 09: `09-lockdown-ping-fails-ssh-works.png`**
> Terminal on `vm-web-01`: ping showing 100% packet loss, then a successful SSH to `10.0.2.4`. This is the core proof of the lab.
>
> **Screenshot 10: `10-direct-ssh-fails.png`**
> Local terminal showing the SSH attempt to `10.0.2.4` timing out.
 
## Result
 
Not yet run. Check these off after deployment. Each maps to a screenshot.
 
- [ ] `vm-db-01` has no public IP (05)
- [ ] Ping from web to DB works before the NSG rules (06)
- [ ] SSH through the web server to the DB works (07)
- [ ] Three NSG rules are in place on the database server (08)
- [ ] After the deny rule, ping fails and SSH still works (09)
- [ ] Direct SSH to the database from the internet fails (10)
After the build, add two or three sentences here on what I saw and anything that surprised me.
 
## Troubleshooting
 
Not yet built. These are issues I expect based on how Azure NSGs and SSH behave. After the build, replace this table with what actually happened.
 
| Symptom | Likely cause | Solution |
|---|---|---|
| Ping fails before any custom rules | `vm-db-01` is in the wrong subnet or VNet | Check the VM's Networking page for `vnet-lab02/snet-db` |
| Cannot SSH straight to `vm-db-01` from my computer | Expected: it has no public IP | Connect through `vm-web-01` |
| `Permission denied (publickey)` | Key permissions too open, or wrong username | Run `chmod 400 key-lab02.pem` and use `azureuser` |
| Locked out of the web VM | SSH rule restricted to an IP that has since changed | Update the source IP in the NSG rule |
| SSH from web to DB fails after lockdown | `Allow-Web-SSH` missing, wrong priority, or wrong source | Confirm priority 100 and source `10.0.1.0/24` |
| Storage or VM size unavailable | Regional capacity | Pick another small size or region |
 
## Cleanup
 
Delete the resource group to remove everything and stop charges:
 
```bash
az group delete --name rg-lab02-manuel --yes --no-wait
```
 
> **Screenshot 11: `11-cleanup.png`** (optional)
> Resource groups list without `rg-lab02-manuel`, or the delete command output.
 
## Key Takeaways
 
- **No public IP is the strongest control.** A server with no public address has no route from the internet at all, regardless of firewall rules.
- **Azure allows all VNet traffic by default.** A private subnet is not isolated until you add an explicit deny.
- **NSG rules run lowest number first.** A custom deny at priority 4000 wins over Azure's built-in allow at 65000.
- **Test what should fail, not only what should work.** The ping failing after lockdown while SSH still works is the actual evidence that the rules do their job.
- **Forward the key, don't copy it.** SSH agent forwarding reaches the private server without leaving a private key on the web server.
## Screenshot guide
 
| File | What it shows | Hide before committing |
|---|---|---|
| `01-vnet-subnets.png` | Both subnets and ranges | Subscription ID |
| `02-web-vm-overview.png` | Public IP, subnet `snet-web` | Public IP |
| `03-web-nsg-ssh-restricted.png` | SSH limited to one source IP | Your IP |
| `04-nginx-default-page.png` | Web tier reachable from the internet | Public IP in address bar |
| `05-db-vm-no-public-ip.png` | No public IP, private IP `10.0.2.4` | Subscription ID |
| `06-ping-web-to-db.png` | Ping replies inside the VNet | Web public IP if visible |
| `07-ssh-jump-to-db.png` | Prompt on `vm-db-01` | Nothing sensitive |
| `08-db-nsg-rules.png` | Rules 100, 110, 4000 | Subscription ID |
| `09-lockdown-ping-fails-ssh-works.png` | Ping blocked, SSH allowed | Web public IP if visible |
| `10-direct-ssh-fails.png` | Timeout from the internet | Nothing sensitive |
| `11-cleanup.png` | Resources deleted | Subscription ID |
 
Blur or crop public IPs, subscription IDs, and your email. Never screenshot the `.pem` file or paste its contents anywhere.
 
---
 
**Author:** Manuel Yannick Armah | **Project:** Secure 2-Tier Web Application on Azure | **Difficulty:** Beginner | **Time to Complete:** ~60 minutes
 
