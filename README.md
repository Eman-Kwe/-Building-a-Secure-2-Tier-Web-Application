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

<img width="618" height="439" alt="Screenshot 2026-09-30 072951" src="https://github.com/user-attachments/assets/98400495-8c13-4f34-a4f1-641938fc4273" />


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

<img width="574" height="434" alt="Screenshot 2026-09-30 075205" src="https://github.com/user-attachments/assets/6039f34a-d05d-4e84-bf9d-cb4afa90eefc" />


<img width="949" height="429" alt="Screenshot 2026-09-30 080419" src="https://github.com/user-attachments/assets/3834f447-2e57-4b64-8dbe-79629b13105a" />


 
### Part 3: Deploy the database server
 
1. Create a second VM named `vm-db-02`, same region, image, and size. The wizard put it in its own resource group, `vm-db-02_group`.
2. Authentication: SSH public key, username `azureuser`, generate a new key pair.
3. On the **Networking** tab, set the virtual network to `vnet-your-name` and the subnet to `snet-db`.
4. Set **Public IP** to **None**. This is the step that keeps the server off the internet.
5. Public inbound ports: choose **None**. I allowed SSH here at first, which made the wizard create an SSH rule open to any source, and I removed it in Part 5.
6. Click **Review + create**, then **Create**.
7. Open `vm-db-02` and note the private IP. Mine was `10.0.2.4`.


<img width="946" height="440" alt="Screenshot 2026-09-30 080305" src="https://github.com/user-attachments/assets/65a9d018-a297-4041-a474-7383022bea4c" />

### Part 4: Test connectivity between tiers
 
Azure's default rules allow all traffic inside a VNet, so this works before any custom rules exist. This guide shows how to connect to an Azure Linux VM over SSH using the private key (`.pem` file) downloaded when the VM was created. The main steps l used WSL (Ubuntu) on Windows. Alternatives for macOS, Linux, and Windows PowerShell are at the end.
 
### Option A: WSL (Ubuntu on Windows)
 
### 1. Find your Windows username
 
```bash
ls /mnt/c/Users
```
 
Ignore the system folders (`Public`, `Default`, `All Users`, and so on). The remaining folder is your Windows username.
 
### 2. Confirm the key is in Downloads
 
```bash
ls /mnt/c/Users/<windows-username>/Downloads | grep -i pem
```
 
You should see your key file, for example `your-key.pem`.
 
### 3. Copy the key into WSL
 
```bash
mkdir -p ~/.ssh
cp /mnt/c/Users/<windows-username>/Downloads/your-key.pem ~/.ssh/
```
 
Copy the key instead of using it from `/mnt/c`. WSL treats files on the Windows drive as readable by everyone, and SSH rejects keys with open permissions.
 
### 4. Lock down the key permissions
 
```bash
chmod 400 ~/.ssh/your-key.pem
```
 
This makes the key read-only for your user, which is what SSH requires.
 
### 5. Get the VM's public IP
 
In the Azure portal, open the VM and copy the **Public IP address** from the Overview page.
 
### 6. Connect
 
```bash
ssh -i ~/.ssh/your-key.pem azureuser@<public-ip>
```
 
- Use the admin username you set at creation if it isn't `azureuser`.
- On the first connection, SSH asks you to confirm the host fingerprint. Type `yes`.
- The prompt changes to `azureuser@<vm-name>:~$`, which means you're inside the VM.
### Option B: macOS or Linux terminal
 
No copy step is needed. Lock down the key where it is and connect:
 
```bash
chmod 400 ~/Downloads/your-key.pem
ssh -i ~/Downloads/your-key.pem azureuser@<public-ip>
```
 
### Option C: Windows PowerShell (no WSL)
 
Windows has OpenSSH built in, but it also rejects keys with open permissions. From the folder holding the key:
 
```powershell
icacls .\your-key.pem /inheritance:r
icacls .\your-key.pem /grant:r "$($env:USERNAME):(R)"
ssh -i .\your-key.pem azureuser@<public-ip>
```

<img width="629" height="326" alt="Screenshot 2026-09-30 083322" src="https://github.com/user-attachments/assets/abd466aa-9b68-462b-bfd9-58d7c8ded5d4" />

<img width="433" height="266" alt="Screenshot 2026-09-30 081937" src="https://github.com/user-attachments/assets/4df48e43-7088-4f40-b4af-a79eaa089470" />
 
### Part 5: Lock down the database server
 
Right now any subnet in the VNet could reach the database. Open `vm-db-02-nsg` → **Inbound security rules** and make three changes, in this order so the SSH path is never cut off.
 
1. **Add the allow rule:**
| Priority | Name | Source | Service / Port | Action |
|---|---|---|---|---|
| 100 | `Allow-Web-SSH` | IP addresses `10.0.1.0/24` | SSH, 22 | Allow |
 
2. **Delete the wide-open rule.** The wizard had created an `SSH` rule at priority 300 with source `Any`. Its low number means Azure checks it before the deny rule, so leaving it would defeat the lockdown.

<img width="661" height="421" alt="Screenshot 2026-09-30 083902" src="https://github.com/user-attachments/assets/255d3ebd-b11a-435d-8dec-924172eadd3c" />

 
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


<img width="956" height="437" alt="Screenshot 2026-09-30 085658" src="https://github.com/user-attachments/assets/92bc177f-6c76-4670-a15a-369df79a99e6" />

## Troubleshooting
 
Everything below happened during this build.
 
| Symptom | Cause | Solution |
|---|---|---|
| Ping from the web VM to the database VM fails | The VMs are in different VNets, the database VM is in the wrong subnet, or an NSG rule blocks ICMP | Confirm both VMs are in the same VNet and the database VM is in its intended subnet (for example `snet-db`). Check that no NSG rule blocks ICMP between the subnets. |
| Can't SSH into the database VM from your home computer | The database VM has no public IP, so it can't be reached directly from the internet | SSH into the web VM first, then connect to the database VM's private IP from there, or use an SSH config file with `ProxyJump`. |
| Ping to the database VM works, but SSH from the web VM says `Permission denied (publickey)` | The network is fine. The web VM doesn't have the database VM's private key, so it has nothing to offer | From your own machine, use an SSH config with `ProxyJump` (recommended), or copy the database key to the web VM, use it, and delete it afterward. Also confirm the admin username and that you're using the key the database VM was created with. |
| `ssh` to the web VM hangs with no output, and a TCP test to port 22 says blocked | The web VM's NSG contains only Azure's default rules and no SSH allow rule, so `DenyAllInBound` drops the connection silently | In the NSG, add an inbound rule allowing TCP port 22 from your own IP (for example `Allow-SSH-MyIP` at priority 300). Get your IPv4 address with `curl -4 ifconfig.me`. A home IP can change, so update the rule if SSH stops working later. |
| Ping to the database VM succeeds at about 0.02 ms | You're still logged into the database VM and are pinging it from itself. Real VM-to-VM latency is around 0.5 to 5 ms | Run `hostname` before testing. The prompt must read `azureuser@<web-vm-name>` when you test the web-to-database path. Type `exit` to leave the database VM. |
 
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
 
 
