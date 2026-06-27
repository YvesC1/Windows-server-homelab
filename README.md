# 🖥️ Windows Server Home Lab — Active Directory Environment

> A hands-on home lab simulating a corporate Active Directory environment using Windows Server 2022 and Windows 11 Pro on Proxmox VE. Built to develop practical IT administration skills in identity management, Group Policy, DNS/DHCP, and PowerShell automation.

---

## 📋 Project Overview

This project demonstrates the end-to-end setup of an on-premises Active Directory environment — the same infrastructure used by thousands of organizations today. The lab mirrors real-world enterprise configurations I encounter and manage in my IT support role.

**Key Skills Demonstrated:**
- Windows Server 2022 installation and configuration
- Active Directory Domain Services (AD DS) setup and management
- DNS and DHCP server configuration
- Organizational Unit (OU) design and Group Policy Objects (GPOs)
- PowerShell automation for bulk user provisioning
- Domain join procedures and client configuration

---

## 🏗️ Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Proxmox VE Host                   │
│                                                     │
│  ┌──────────────────┐      ┌──────────────────┐    │
│  │      DC01        │      │    CLIENT01      │    │
│  │  Windows Server  │      │  Windows 11 Pro  │    │
│  │      2022        │      │                  │    │
│  │                  │      │  • Domain joined │    │
│  │  • AD DS         │      │  • AD user login │    │
│  │  • DNS Server    │◄────►│  • GPO applied   │    │
│  │  • DHCP Server   │      │                  │    │
│  │  • Domain: corp.local   │                  │    │
│  │  • IP: 192.168.1.1      │  IP: DHCP        │    │
│  └──────────────────┘      └──────────────────┘    │
│        │       │                    │               │
│     [vmbr0] [vmbr1]─────────────[vmbr1]            │
│        │       (isolated AD network)                │
│     Internet                                        │
└─────────────────────────────────────────────────────┘
```

**Hypervisor:** Proxmox VE (existing home lab)  
**Network:** Isolated Linux bridge (`vmbr1`) — `192.168.1.0/24`  
**Domain:** `corp.local`  
**Domain Controller IP (static):** `192.168.1.1`  
**Client IP:** DHCP-assigned from DC (`192.168.1.100–200`)

---

## 🛠️ Technologies Used

| Technology | Purpose |
|---|---|
| Proxmox VE | Hypervisor for running VMs |
| Windows Server 2022 (Eval) | Domain Controller OS |
| Windows 11 Pro | Client workstation OS |
| Active Directory Domain Services | Identity & access management |
| DNS Server | Name resolution for the domain |
| DHCP Server | Automatic IP assignment for clients |
| PowerShell | Bulk user provisioning automation |
| Group Policy Management Console | Policy enforcement across domain |

---

## 📁 Repository Structure

```
windows-server-homelab/
├── README.md                    ← You are here
├── screenshots/
│   ├── 01-proxmox-vms.png
│   ├── 02-ad-ds-install.png
│   ├── 03-domain-promotion.png
│   ├── 04-ou-structure.png
│   ├── 05-aduc-users.png
│   ├── 06-gpo-config.png
│   ├── 07-client-domain-join.png
│   └── 08-domain-login-verified.png
├── scripts/
│   └── bulk-create-users.ps1   ← PowerShell script for user provisioning
└── docs/
    └── implementation-notes.md ← Configuration notes and lessons learned
```

---

## 🚀 Implementation Steps

### Phase 1 — VM Setup

Created two virtual machines in Proxmox VE:

**DC01 (Domain Controller)**
- OS: Windows Server 2022 Standard Evaluation (Desktop Experience)
- RAM: 4096 MB | Disk: 50 GB
- Adapter 1: NAT (internet)
- Adapter 2: Internal Network (`AD_LAB`)

**CLIENT01**
- OS: Windows 11 Pro
- RAM: 2048 MB | Disk: 50 GB
- TPM 2.0 emulation enabled in Proxmox (required for Windows 11)
- Adapter: Internal bridge (`vmbr1`)

---

### Phase 2 — Domain Controller Configuration

**Static IP assigned to the internal adapter:**
```
IP Address:   192.168.1.1
Subnet Mask:  255.255.255.0
DNS:          127.0.0.1 (self)
```

**Roles installed via Server Manager:**
- Active Directory Domain Services
- DNS Server
- DHCP Server

**Domain created:** `corp.local` (new forest)

**DHCP Scope configured:**
- Range: `192.168.1.100` – `192.168.1.200`
- DNS option pointing to `192.168.1.1`

---

### Phase 3 — Active Directory Structure

**Organizational Unit (OU) Design:**
```
corp.local
├── _IT
├── _HR
├── _Finance
├── _Computers
└── _ServiceAccounts
```

**Users Created:**
- 3 users created manually via ADUC to practice the UI workflow
- 50 users bulk-created via PowerShell script (see `/scripts/`)

**Security Groups Created:**
- `IT-Admins` → IT OU
- `HelpDesk` → IT OU
- `Finance-Users` → Finance OU
- `Standard-Users` → HR OU

---

### Phase 4 — Group Policy

**GPO: Standard User Restrictions** (linked to `_HR` OU)
- Enforce desktop wallpaper
- Disable access to Control Panel
- Set minimum password length to 10 characters

**Verified:** Ran `gpupdate /force` on CLIENT01 after domain join — policies applied successfully.

---

### Phase 5 — Client Domain Join

1. Set CLIENT01 DNS to `192.168.1.1`
2. System Properties → Computer Name → Change → Domain: `corp.local`
3. Provided Domain Admin credentials
4. Rebooted → logged in with a domain user account

**Verification commands run on CLIENT01:**
```powershell
whoami          # Returns: corp\username
ping dc01       # Resolves and replies
gpresult /r     # Confirms GPO applied

__

## 🧠 Key Takeaways

**What I reinforced through this lab:**

- The relationship between DNS and Active Directory — AD won't function without proper DNS, which is why the DC is its own DNS server and clients must point to it
- How OU design affects Group Policy scope — GPOs apply at the OU level and inherit downward
- Why `Windows 11 Pro` is required for domain join — the Home edition lacks this capability
- PowerShell as a force multiplier — what takes 30 minutes clicking through ADUC takes 30 seconds with a script
- The importance of VM snapshots — being able to roll back saved hours of re-configuration

**How this connects to real work:**

In my IT support role at a nonprofit organization, I work daily with Microsoft 365/Entra ID, which is the cloud evolution of on-premises Active Directory. This lab deepened my understanding of the foundational concepts (OUs, GPOs, domain joins, DNS, DHCP) that underpin both on-prem AD and Entra ID hybrid environments.

---

## 📸 Screenshots

> *(Add your screenshots to the `/screenshots/` folder as you complete each phase)*

| Step | Screenshot |
|---|---|
| Proxmox — Both VMs running | `screenshots/01-proxmox-vms.png` |
| AD DS Role installation | `screenshots/02-ad-ds-install.png` |
| Domain promotion wizard | `screenshots/03-domain-promotion.png` |
| OU structure in ADUC | `screenshots/04-ou-structure.png` |
| User list in ADUC | `screenshots/05-aduc-users.png` |
| GPO configuration | `screenshots/06-gpo-config.png` |
| CLIENT01 domain join | `screenshots/07-client-domain-join.png` |
| Domain login verified | `screenshots/08-domain-login-verified.png` |

---

## 📚 Resources Used

- [YouTube Playlist — Windows Server Home Lab Project](https://youtube.com/playlist?list=PLAdEnQWAAbfXMY2D4HVZOe-ChfTKmaJfQ)
- [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/) — Windows Server 2022 ISO
- [Windows 11 Pro ISO](https://www.microsoft.com/software-download/windows11)
- [Proxmox VE](https://www.proxmox.com/en/downloads)
- Microsoft Learn — Active Directory Documentation

---

## 👤 About

**Yves** | IT Support Specialist | CompTIA A+, Network+, Security+  
Working toward CCNA and FAA AMT certification  
[GitHub Profile](https://github.com/yourusername) | [LinkedIn](https://linkedin.com/in/yourprofile)
