# Secure 2-Tier Web Application on Azure

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
- Reaching a private server through a jump host, using SSH agent forwarding instead of copying keys
- Validating security controls by testing what should fail as well as what should work
- Troubleshooting Azure portal and SSH connectivity issues from error messages
- Resource group organization and cost control

## Architecture

<img width="859" height="620" alt="architecture (2)" src="https://github.com/user-attachments/assets/02436f7e-e5d2-486a-8030-e9dfca2d6f87" />


 
- The internet can reach only `vm-web-01` (HTTP 80, and SSH 22 from my IP).
- `vm-db-01` has no public IP. It accepts SSH and database traffic only from the web subnet `10.0.1.0/24`.
- Editable source: [`docs/architecture.drawio`](docs/architecture.drawio)
## Prerequisites
 
- [ ] Active Azure subscription (Free Tier works)
- [ ] Terminal with an SSH client (I used WSL Ubuntu on Windows)
- [ ] Basic comfort with Linux commands and SSH keys
## Naming Conventions
 
| Resource | Name |
|---|---|
| Resource group (network + web VM) | `rglab02-your-name` |
| Resource group (database VM, created by the portal wizard) | `vm-db-02_group` |
| VNet | `vnet-your-name` (`10.0.0.0/16`) |
| Public subnet | `snet-web` (`10.0.1.0/24`) |
| Private subnet | `snet-db` (`10.0.2.0/24`) |
| Web VM | `vm-web-01` |
| Database VM | `vm-db-02` |
| SSH keys | `your-web-key` (web), `your-db-key` (database) |
| VM size | `Standard_D2als_v7` (about $0.08 per hour) |
| Image | Ubuntu Server 24.04 LTS |

Region for everything: East US.
 
## Project Steps
 
### Part 1: Build the network
1. In the Azure portal, search for **Virtual networks** and click **Create**.
2. Create the resource group `rglab02-your-name` and name the VNet `vnet-your-name`, region East US.
3. On the **Address space** tab, keep the single address space `10.0.0.0/16`. Do not add other address spaces for the subnets.
4. Delete the `default` subnet. Click **+ Add a subnet** and create `snet-web` (starting address `10.0.1.0`, size `/24`), then `snet-db` (starting address `10.0.2.0`, size `/24`). Leave **Enable private subnet** checked on both.
5. Click **Review + create**, then **Create**.

<img width="1511" height="693" alt="preview (17)" src="https://github.com/user-attachments/assets/fc02aa08-3bec-4d51-9295-f98b41d37a36" />

### Part 2: Deploy the web server
 
1. **Virtual machines** → **Create** → **Azure virtual machine**.
2. Resource group `rglab02-your-name`, name `vm-web-01`, region East US.
3. Image: Ubuntu Server 24.04 LTS. Size: click **See all sizes** and pick a small size that is available to your subscription (see Troubleshooting for why I used `Standard_D2als_v7`).
4. Authentication: SSH public key, username `azureuser`, generate a new key pair. Download the `.pem` when prompted. It is shown only once.
5. On the **Networking** tab, set the virtual network to `vnet-your-name` and the subnet to `snet-web`. Create a new public IP (Standard).
6. Do the Basics tab first and Networking last. Changing the resource group or region afterward resets the network selection.
7. Click **Review + create**, then **Create**.
8. After deployment, open `vm-web-01-nsg` → **Inbound security rules**. The wizard did not create the SSH rule for me, so I added one:
   - **Source:** IP Addresses, my IPv4 address (`curl -4 ifconfig.me` prints it)
   - **Service:** SSH (port 22, TCP)
   - **Action:** Allow
   - **Priority:** 300
   - **Name:** `Allow-SSH-MyIP`
![Web VM overview](docs/screenshots/02-web-vm-overview.png)
 
![Web VM NSG with SSH limited to one source IP](docs/screenshots/03-web-nsg-ssh-restricted.png)
 
### Part 3: Deploy the database server
 
1. Create a second VM named `vm-db-02`, same region, image, and size. The wizard put it in its own resource group, `vm-db-02_group`.
2. Authentication: SSH public key, username `azureuser`, generate a new key pair.
3. On the **Networking** tab, set the virtual network to `vnet-your-name` and the subnet to `snet-db`.
4. Set **Public IP** to **None**. This is the step that keeps the server off the internet.
5. Public inbound ports: choose **None**. I allowed SSH here at first, which made the wizard create an SSH rule open to any source, and I removed it in Part 5.
6. Click **Review + create**, then **Create**.
7. Open `vm-db-02` and note the private IP. Mine was `10.0.2.4`.
![Database VM with no public IP](docs/screenshots/05-db-vm-no-public-ip.png)
 
### Part 4: Test connectivity between tiers
 
Azure's default rules allow all traffic inside a VNet, so this works before any custom rules exist.
 
1. Copy the keys into WSL and lock down their permissions. Windows folders cannot hold a key with strict permissions:
```bash
mkdir -p ~/.ssh
cp /mnt/c/Users/<windows-user>/Downloads/your-web-key.pem ~/.ssh/
cp /mnt/c/Users/<windows-user>/Downloads/your-db-key.pem ~/.ssh/
chmod 400 ~/.ssh/your-web-key.pem ~/.ssh/your-db-key.pem
```
 
2. Load both keys into the SSH agent:
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/your-web-key.pem
ssh-add ~/.ssh/your-db-key.pem
```
 
3. Connect to the web server with agent forwarding, so the private keys never sit on it:
```bash
ssh -A -i ~/.ssh/your-web-key.pem azureuser@<web-public-ip>
```
 
4. From `vm-web-01`, ping the database server and then jump to it:
```bash
ping -c 4 10.0.2.4
ssh azureuser@10.0.2.4
hostname
```
 
![Ping from the web VM to the database VM](docs/screenshots/06-ping-web-to-db.png)
 
![SSH jump to the database VM](docs/screenshots/07-ssh-jump-to-db.png)
 
### Part 5: Lock down the database server
 
Right now any subnet in the VNet could reach the database. Open `vm-db-02-nsg` → **Inbound security rules** and make three changes, in this order so the SSH path is never cut off.
 
1. **Add the allow rule:**
| Priority | Name | Source | Service / Port | Action |
|---|---|---|---|---|
| 100 | `Allow-Web-SSH` | IP addresses `10.0.1.0/24` | SSH, 22 | Allow |
 
2. **Delete the wide-open rule.** The wizard had created an `SSH` rule at priority 300 with source `Any`. Its low number means Azure checks it before the deny rule, so leaving it would defeat the lockdown.
3. **Add the deny rule:**
| Priority | Name | Source | Service / Port | Action |
|---|---|---|---|---|
| 4000 | `Deny-VNet-Other` | Service tag `VirtualNetwork` | Custom, any port, any protocol | Deny |
 
The deny rule at 4000 is checked before Azure's built-in "allow VNet" rule at 65000, so anything not explicitly allowed is dropped. If you install a database, add an allow rule from `10.0.1.0/24` to its port (3306 or 5432) below priority 4000.
 
![Database NSG rules](docs/screenshots/08-db-nsg-rules.png)
 
Then test again from `vm-web-01`:
 
```bash
clear
ping -c 4 -W 2 10.0.2.4
ssh azureuser@10.0.2.4 hostname
```
 
![Ping blocked, SSH still allowed](docs/screenshots/09-lockdown-ping-fails-ssh-works.png)
 
Last, from my own computer (not the web VM):
 
```bash
ssh -o ConnectTimeout=10 -i ~/.ssh/your-db-key.pem azureuser@10.0.2.4
```
 
![Direct SSH from the internet times out](docs/screenshots/10-direct-ssh-fails.png)
 
## Result
 
| Test | Expected | Actual |
|---|---|---|
| Database VM has a public IP | No | No, and its private IP is `10.0.2.4` |
| Ping web → database, before NSG rules | Replies | 4 of 4 replies, 0% loss, about 3.6 ms average |
| SSH web → database, before NSG rules | Works | Worked, prompt `azureuser@vm-db-02` |
| Ping web → database, after NSG rules | Blocked | 100% packet loss |
| SSH web → database, after NSG rules | Works | Worked, `hostname` printed `vm-db-02` |
| SSH to `10.0.2.4` from the internet | Fails | `Connection timed out` |
| Database VM reaches the internet | No | No (`changelogs.ubuntu.com` was unreachable from it) |
 
The before/after ping is the clearest evidence. The same ping that got four replies returned 100% loss once the deny rule was in place, while SSH kept working because the allow rule at priority 100 matched first.
 
## Troubleshooting
 
Everything below happened during this build.
 
| Symptom | Cause | Solution |
|---|---|---|
| Red X on the Address space tab: "Address prefix 10.0.0.0/16 overlaps with 10.0.1.0/24, 10.0.2.0/24" | I added the subnet ranges as separate address spaces instead of subnets. Address spaces in one VNet cannot overlap | Deleted the extra address spaces, kept `10.0.0.0/16`, and created the ranges with **+ Add a subnet** |
| VM size error: "NotAvailableForSubscription" for `Standard_D2s_v3`; B1s, B1ls, B1ms, and B2s were greyed out | My subscription could not use the small B-series sizes in East US | Used **See all sizes** to find a size that was selectable: `Standard_D2als_v7`, about $0.08 per hour |
| Could not select `snet-web` on the VM's Networking tab. It offered only a new `172.16.0.0/24` subnet | The wizard had reset the network to a new default VNet when I changed settings on Basics | Re-selected `vnet-your-name` in the Virtual network dropdown, and then `snet-web` appeared |
| `ssh` to the web VM hung with no output; a TCP test to port 22 said "blocked" | The web VM's NSG contained only Azure's three default rules. The SSH rule the wizard should have created was missing, so `DenyAllInBound` dropped my connection | Added `Allow-SSH-MyIP` at priority 300, using the IPv4 address from `curl -4 ifconfig.me` |
| Ping to `10.0.2.4` succeeded at 0.02 ms after the lockdown | I was still logged into the database VM, so I was pinging it from itself | Ran `hostname` before testing. The prompt must read `azureuser@vm-web-01` when testing the web-to-database path |
| Extra resource groups in the portal | The wizard created `vm-db-02_group` for the database VM, and an abandoned first attempt left an empty `vm-your-name_group` | Listed all groups before cleanup and deleted all three |
 
## Cleanup
 
Delete every resource group this build created. Deleting only one leaves the other VM billing
 
## Key Takeaways
 
- **No public IP is the strongest control.** A server with no public address has no route from the internet, whatever the firewall rules say. The timed-out SSH from outside shows it.
- **Azure allows all VNet traffic by default.** A private subnet is not isolated until you add an explicit deny.
- **NSG rules run lowest number first.** My deny at priority 4000 beats Azure's built-in allow at 65000, but a leftover wizard rule at 300 would have beaten my deny. Check the whole list, not just the rules you added.
- **Test what should fail, not only what should work.** Ping failing while SSH still worked is the evidence the rules do their job.
- **Do not trust the wizard to create what you expect.** Mine skipped the web VM's SSH rule and added a wide-open one to the database VM. Open the NSG and read it.
- **Run each test from the right machine.** A ping that succeeds from the wrong host proves nothing, so confirm the prompt first.
- **Forward the key, don't copy it.** SSH agent forwarding reaches the private server without leaving a private key on the web server.
---
 
**Author:** Manuel Yannick Armah 

**Project:** Secure 2-Tier Web Application on Azure 

**Difficulty:** Beginner 

**Time to Complete:** 60 minutes
 
 
