# 🚀 Enterprise Virtual Machine Lab: Proxmox VE, Windows Server 2022 & Windows 11 Pro

[![Proxmox VE](https://img.shields.io/badge/Hypervisor-Proxmox%20VE%208.x-E65100?style=for-the-badge&logo=proxmox&logoColor=white)](https://www.proxmox.com/)
[![Windows Server](https://img.shields.io/badge/Domain%20Controller-Windows%20Server%202022-0078D4?style=for-the-badge&logo=windows-server&logoColor=white)](https://www.microsoft.com/evalcenter/evaluate-windows-server-2022)
[![Windows 11](https://img.shields.io/badge/Client%20OS-Windows%2011%20Pro-0078D4?style=for-the-badge&logo=windows11&logoColor=white)](https://www.microsoft.com/software-download/windows11)
[![License](https://img.shields.io/badge/License-MIT-green.svg?style=for-the-badge)](LICENSE)
[![Cost](https://img.shields.io/badge/Cost-%240%20(Open%20Source%20%2B%20Eval)-brightgreen?style=for-the-badge)](#-cost--licensing)

A comprehensive, step-by-step engineering guide for building an isolated, production-grade Active Directory Domain Services (AD DS) lab environment. This project covers setting up a bare-metal **Proxmox VE** hypervisor, deploying a **Windows Server 2022** Domain Controller with DHCP and software RAID 5, and configuring a **Windows 11 Pro** client VM joined to the domain with mirrored storage.

---

## 📋 Table of Contents

- [Architecture & Topology](#-architecture--topology)
- [Environment Specifications](#-environment-specifications)
- [Cost & Licensing](#-cost--licensing)
- [Part 1: Set Up Proxmox VE Hypervisor](#️-part-1-set-up-proxmox-ve-hypervisor)
  - [1.1 Download & Verify ISOs](#11-download--verify-isos)
  - [1.2 Create Bootable USB Media](#12-create-bootable-usb-media)
  - [1.3 Bare-Metal Installation](#13-bare-metal-installation)
  - [1.4 Post-Install: Free Repository Setup](#14-post-install-free-repository-setup)
  - [1.5 Upload ISOs to Storage](#15-upload-isos-to-storage)
- [Part 2: Deploy Windows Server 2022 (Domain Controller)](#-part-2-deploy-windows-server-2022-domain-controller)
  - [2.1 VM Creation (OVMF / UEFI + TPM 2.0)](#21-vm-creation-ovmf--uefi--tpm-20)
  - [2.2 OS Installation & Initial Credentials](#22-os-installation--initial-credentials)
  - [2.3 Network Isolation (`vmbr1`)](#23-network-isolation-vmbr1)
  - [2.4 Host Name, Time Zone & Static IP Setup](#24-host-name-time-zone--static-ip-setup)
  - [2.5 Install & Configure AD DS (Forest Setup)](#25-install--configure-ad-ds-forest-setup)
  - [2.6 Structure Organizational Units & Users](#26-structure-organizational-units--users)
  - [2.7 Deploy & Authorize DHCP Role](#27-deploy--authorize-dhcp-role)
  - [2.8 Configure DHCP Scope & Exclusions](#28-configure-dhcp-scope--exclusions)
  - [2.9 Storage Subsystem: Software RAID 5 Setup](#29-storage-subsystem-software-raid-5-setup)
- [Part 3: Deploy Windows 11 Pro (Domain Client)](#-part-3-deploy-windows-11-pro-domain-client)
  - [3.1 VM Creation (OVMF / UEFI + TPM 2.0)](#31-vm-creation-ovmf--uefi--tpm-20)
  - [3.2 OS Installation & Profile Configuration](#32-os-installation--profile-configuration)
  - [3.3 Network Attachment (`vmbr1`)](#33-network-attachment-vmbr1)
  - [3.4 Join Active Directory Domain](#34-join-active-directory-domain)
  - [3.5 Domain Logon Verification](#35-domain-logon-verification)
  - [3.6 Storage Subsystem: Software RAID 0 (Mirrored Volume)](#36-storage-subsystem-software-raid-0-mirrored-volume)
- [Technical Reference & Best Practices](#-technical-reference--best-practices)
- [License](#-license)

---

## 📐 Architecture & Topology

```text
                               +-------------------------------------------------+
                               |             Proxmox VE Hypervisor Host          |
                               |             FQDN: pve.lab.local                 |
                               |             Management IP: https://<IP>:8006    |
                               +------------------------+------------------------+
                                                        |
                                                        | Physical Bridge (vmbr0)
                                                        v
                                             +--------------------+
                                             | Production Network |
                                             +--------------------+
                                                        |
                                                        | Internal Isolated Bridge (vmbr1)
                          +-----------------------------+-----------------------------+
                          |                                                           |
                          v                                                           v
   +---------------------------------------------+             +---------------------------------------------+
   |   Windows Server 2022 (VM ID: WinSer22)     |             |     Windows 11 Pro Client (VM ID: Win11Pro)  |
   +---------------------------------------------+             +---------------------------------------------+
   | - Role: Active Directory Domain Controller   |             | - Role: Domain-Joined Client Workstation    |
   | - Hostname: DC1                             |             | - Hostname: Win11-Client                    |
   | - Domain: yourname.local                    |  DHCP / DNS | - IP Address: Dynamic (via DHCP Scope1)     |
   | - Static IP: 192.168.0.10 /24               |<----------->| - DNS: 192.168.0.10                         |
   | - Roles: AD DS, DNS, DHCP Server            |             | - Security: UEFI, TPM 2.0 Enabled           |
   | - Security: UEFI, TPM 2.0 Enabled           |             | - Storage: System Disk + Mirrored (2x 5GB)  |
   | - Storage: System Disk + RAID 5 (3x 5GB)    |             +---------------------------------------------+
   +---------------------------------------------+
```

## 📊 Environment Specifications

| Parameter | Proxmox VE Host | Windows Server 2022 (DC1) | Windows 11 Pro Client |
|---|---|---|---|
| **Role** | Bare-Metal Hypervisor | Domain Controller / DHCP / DNS | Domain-Joined Client Workstation |
| **BIOS Mode** | UEFI / Legacy BIOS | OVMF (UEFI) + TPM 2.0 | OVMF (UEFI) + TPM 2.0 |
| **vCPU Cores** | Host Hardware (Min 2 Cores) | 2 vCPU | 2 vCPU |
| **RAM** | 8 GB – 16 GB+ | 4096 MB | 4096 MB |
| **Primary Storage** | 32 GB+ (ext4/ZFS) | 50 GB (Thin Provisioned) | 64 GB (Thin Provisioned) |
| **Data Storage** | – | 3 x 5 GB Virtual Disks (RAID 5) | 2 x 5 GB Virtual Disks (RAID 1 Mirror) |
| **Network Interface** | `vmbr0` (Physical Management) | `vmbr1` (Internal Isolated) | `vmbr1` (Internal Isolated) |
| **IP Allocation** | Static Management IP | Static (192.168.0.10/24) | Dynamic (192.168.0.11–.254) |

## 💰 Cost & Licensing

> **Note**
> Proxmox VE is free and open-source software under the GNU AGPLv3 license. This complete lab infrastructure can be deployed at $0 licensing cost.

- **Proxmox VE:** Fully functional without subscription. Access to the `pve-no-subscription` repository is free forever.
- **Windows Server 2022 & Windows 11 Pro:** Deployed using Microsoft Evaluation ISOs, providing full feature functionality for testing and training.
- **Hardware Efficiency:** Ideal for non-profits, homelabs, or corporate training labs looking to consolidate server hardware without per-seat hypervisor licensing fees.

---

## 🛠️ Part 1: Set Up Proxmox VE Hypervisor

### 1.1 Download & Verify ISOs

1. Download the latest official Proxmox VE installation ISO from the [Proxmox Downloads Page](https://www.proxmox.com/en/downloads).
2. Verify the downloaded ISO checksum:

```bash
# Linux/macOS
sha256sum proxmox-ve_*.iso
```

```powershell
# Windows PowerShell
Get-FileHash -Algorithm SHA256 .\proxmox-ve_*.iso
```

### 1.2 Create Bootable USB Media

Flash the ISO image onto an 8 GB+ USB flash drive:

- **Rufus (Windows):** Select target USB → Partition scheme: **GPT** → Target system: **UEFI**.

> **Important**
> When prompted by Rufus, select **Write in DD Image mode**. Proxmox uses a hybrid ISO structure that requires raw sector writing.

- **dd (Linux/macOS):**

```bash
sudo dd if=proxmox-ve.iso of=/dev/sdX bs=1M status=progress conv=fsync
```

### 1.3 Bare-Metal Installation

1. Insert the bootable USB into the target machine and enter system BIOS/UEFI. Verify that hardware virtualization extensions (Intel VT-x or AMD-V) are enabled.
2. Boot from the USB stick and select **Install Proxmox VE (Graphical)**.
3. Accept the End User License Agreement (EULA).
4. **Target Harddisk:** Select target install disk. Click **Options** to select the filesystem:
   - **ext4 (LVM):** Recommended default for single-drive setups.
   - **ZFS (RAID1/10/Z):** Recommended only if multi-drive redundancy and extra system RAM (8 GB+) are available.
5. **Location and Timezone:** Set country, timezone, and keyboard layout.
6. **Password and Email:** Specify root administration password and email notification address.
7. **Management Network Configuration:**
   - Management Interface: Select physical NIC.
   - Hostname (FQDN): Set full domain name (e.g., `pve.lab.local`). Avoid `.local` as the root domain if mDNS conflicts are expected.
   - IP Address / Netmask / Gateway / DNS: Set static management IP parameters.
8. Complete installation and reboot. Log into the web administration UI at `https://<your-proxmox-ip>:8006`.

### 1.4 Post-Install: Free Repository Setup

To update Proxmox VE without an enterprise subscription:

1. Navigate to **Node → Repositories**.
2. Disable the `pve-enterprise` repository.
3. Click **Add → No-Subscription repository** (`pve-no-subscription`).
4. Go to **APT → Refresh → Upgrade** to pull the latest system packages.

### 1.5 Upload ISOs to Storage

1. Download ISO files for:
   - Windows Server 2022 Evaluation (Microsoft Evaluation Center)
   - Windows 11 Pro (Microsoft Windows 11 Download Page)
   - **VirtIO drivers ISO** (`virtio-win.iso`) from the [Fedora VirtIO driver repository](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/) — Proxmox's default disk controller (VirtIO SCSI) isn't natively recognized by Windows Setup, so this ISO is required for the installer to detect the virtual hard disk.
2. In the Proxmox web UI: navigate to **Server Node → local (pve) → ISO Images**.
3. Click **Upload** and upload the Windows Server 2022, Windows 11, and VirtIO driver ISOs.

---

## 💻 Part 2: Deploy Windows Server 2022 (Domain Controller)

### 2.1 VM Creation (OVMF / UEFI + TPM 2.0)

1. Click **Create VM** in the top right header.
2. **General tab:** VM ID: auto-assigned or custom (e.g., `100`) · Name: `WinSer22`
3. **OS tab:** Media: **Use CD/DVD disc image file (ISO)** · Storage: `local` · ISO Image: Windows Server 2022 ISO · Type: **Microsoft Windows** · Version: **11/2022/2025**
4. **System tab:** Graphic card: Default · BIOS: **OVMF (UEFI)** · EFI Storage: `local-lvm` · Machine: `q35` (or default `i440fx`) · Pre-enroll keys: checked · Add TPM: checked (v2.0, Storage: `local-lvm`)
5. **Disks tab:** Storage: `local-lvm` · Disk size: **50 GB** · Format: Raw disk image (or qcow2) · Bus/Device: **VirtIO SCSI** (default)
6. **CPU tab:** Cores: 2 (minimum recommended)
7. **Memory tab:** Memory: **4096 MB** · Uncheck "Ballooning Device" (predictable RAM allocation is preferred for AD DS integrity)
8. **Network tab:** Bridge: `vmbr0` (temporary for setup; isolated in step 2.3)
9. Review the **Confirm** summary and click **Finish** (don't start the VM yet).
10. Select `WinSer22` → **Hardware → Add → CD/DVD Drive**. Storage: `local` · ISO Image: `virtio-win.iso`. This attaches the VirtIO drivers as a second, separate CD/DVD drive alongside the Windows Server ISO.

### 2.2 OS Installation & Initial Credentials

1. Select VM `WinSer22` → **Start** → **Console** (noVNC).
2. Press any key when prompted to boot from CD/DVD.
3. **Setup Configuration:** choose language/time format → **Next** → **Install Now**.
4. **Edition:** select **Windows Server 2022 Standard Evaluation (Desktop Experience)**.
5. Accept license terms → **Custom: Install Windows only (advanced)**.
6. On the "Where do you want to install Windows?" screen, **no drives will be listed** — this is expected with a VirtIO SCSI controller. Click **Load driver**.
7. Uncheck "Hide drivers that aren't compatible with this computer's hardware," then click **Browse**.
8. Navigate to the VirtIO CD drive → `vioscsi` → your Windows Server version folder (e.g. `2k22`) → `amd64`, then click **OK**.
9. Select the listed **Red Hat VirtIO SCSI controller** driver → **Next** to load it. The 50 GB virtual disk should now appear.
10. Select the unallocated space (50 GB) → **Next** to begin installing.
11. Upon reboot, set the built-in Administrator password: `Password1`.
12. Click **Finish** and log in via Console (Ctrl+Alt+Del button at top-left of noVNC).

> **Tip**
> Always shut down Windows guests cleanly from inside the OS (Start → Power → Shut down) rather than using Proxmox's hard **Stop** button, to avoid filesystem corruption.

### 2.3 Network Isolation (`vmbr1`)

To prevent lab traffic (DHCP, AD broadcast) from polluting the host/production network:

1. In the Proxmox UI, navigate to **Datacenter → Node → System → Network**.
2. Click **Create → Linux Bridge**.
   - Name: `vmbr1`
   - IPv4/CIDR: leave blank
   - Gateway: leave blank
   - Bridge ports: leave blank (no physical NIC attached)
   - Comment: `Internal Lab Network`
3. Click **Create → Apply Configuration**.
4. Go to VM `WinSer22` → **Hardware** → double-click **Network Device** (`net0`).
5. Change Bridge to `vmbr1` → **OK**.

### 2.4 Host Name, Time Zone & Static IP Setup

**Set Time Zone**
1. Open Server Manager → select **Local Server**.
2. Click the current Time zone link → **Change time zone…**.
3. Select **(UTC-05:00) Eastern Time (US & Canada)** → **OK**.

**Configure Static Network Interface**
1. In Server Manager → **Local Server**, click the network adapter link next to the IPv4 address.
2. Right-click the network interface → **Properties**.
3. Uncheck **Internet Protocol Version 6 (TCP/IPv6)**.
4. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
5. Configure static parameters:
   - IP address: `192.168.0.10`
   - Subnet mask: `255.255.255.0`
   - Default gateway: leave blank
   - Preferred DNS server: `192.168.0.10` (self-referencing for AD DS)
6. Click **OK → Close**.

**Rename Computer Hostname**
1. Server Manager → **Local Server** → click the Computer name link.
2. In System Properties → **Change…**.
3. Computer Name: `DC1` → **OK**.
4. Restart the virtual machine when prompted.

### 2.5 Install & Configure AD DS (Forest Setup)

1. Log back into `DC1` as Administrator (`Password1`).
2. Server Manager opens automatically → **Manage → Add Roles and Features**.
3. **Installation Type:** Role-based or feature-based installation → **Next**.
4. **Server Selection:** ensure `DC1` is selected → **Next**.
5. **Server Roles:** check **Active Directory Domain Services**. When prompted, click **Add Features** → **Next**.
6. **Features & AD DS pages:** click **Next** through defaults.
7. **Confirmation:** check "Restart the destination server automatically if required" → **Install**.

**Promote Server to Domain Controller**
1. Once installation completes, click the notification flag → **Promote this server to a domain controller**.
2. **Deployment Configuration:** select **Add a new forest** → Root domain name: `yourname.local` → **Next**.
3. **Domain Controller Options:** Domain Functional Level: Windows Server 2016 · Password/DSRM Password: `Password1` → **Next**.
4. **DNS Options:** **Next** (ignore delegation warning).
5. **Additional Options:** verify NetBIOS domain name (`YOURNAME`) → **Next**.
6. **Paths / Review:** leave default database/log/SYSVOL paths → **Next**.
7. **Prerequisites Check:** ensure checks pass → **Install**.
8. The server restarts automatically. Log back in as `YOURNAME\Administrator` with password `Password1`.

### 2.6 Structure Organizational Units & Users

1. Open Server Manager → **Tools → Active Directory Users and Computers**.
2. Expand domain node `yourname.local`.
3. **Create main OU:** right-click `yourname.local` → **New → Organizational Unit** → Name: `Departments` → **OK**.
4. **Create sub-OUs:** right-click `Departments` → **New → Organizational Unit** for each of: `Sales`, `HR`, `Marketing`.
5. **Provision user accounts:** right-click `Sales` OU → **New → User**:
   - First name: `User1` · User logon name: `user1` → **Next**
   - Password: `Password1`; uncheck "User must change password at next logon"; check "Password never expires" → **Next → Finish**
6. Repeat for `User2` in the `HR` OU and `User3` in the `Marketing` OU.

### 2.7 Deploy & Authorize DHCP Role

1. Server Manager → **Manage → Add Roles and Features** → **Next**.
2. Select **Role-based or feature-based installation** → select `DC1`.
3. **Server Roles:** check **DHCP Server** → **Add Features** → **Next**.
4. Click **Next** through Features → **Install**.
5. **Post-Deployment Configuration:** click the notification flag → **Complete DHCP configuration**.
6. Click **Next** on the Description page.
7. **Authorization:** select "Use the target computer's credentials" (`YOURNAME\Administrator`) → **Commit**.
8. Click **Close**.

### 2.8 Configure DHCP Scope & Exclusions

1. Open Server Manager → **Tools → DHCP**.
2. Expand `dc1.yourname.local` → right-click **IPv4**.

> **Note**
> If the IPv4 icon shows a red status, right-click **IPv4** → **Authorize**, then **Refresh**.

3. Right-click **IPv4** → **New Scope…** → **Next**.
4. Scope Name: `Scope1` → **Next**.
5. **IP Address Range:** Start `192.168.0.1` · End `192.168.0.254` · Length `24` · Subnet Mask `255.255.255.0` → **Next**.
6. **Add Exclusions:** Start IP `192.168.0.1` · End IP `192.168.0.10` → **Add → Next**.
7. **Lease Duration:** keep default (8 days) → **Next**.
8. **Configure DHCP Options:** select "Yes, I want to configure these options now" → **Next**.
9. **Router (Default Gateway):** **Next** (leave blank for the isolated network).
10. **Domain Name and DNS Servers:** confirm parent domain is `yourname.local` and IP `192.168.0.10` is listed → **Next**.
11. **WINS Servers:** **Next**.
12. **Activate Scope:** select "Yes, I want to activate this scope now" → **Next → Finish**.

### 2.9 Storage Subsystem: Software RAID 5 Setup

Demonstrates Windows Server software storage redundancy using dynamic disks:

1. In the Proxmox UI, shut down VM `WinSer22`.
2. Select `WinSer22` → **Hardware → Add → Hard Disk**. Storage: `local-lvm` · Disk Size: `5 GB` → **Add**.
3. Repeat two more times so three extra 5 GB virtual disks are attached (`scsi1`, `scsi2`, `scsi3`).
4. Boot `WinSer22` and log in as Administrator.
5. Right-click Start Menu → **Disk Management**.
6. The Initialize Disk dialog prompts automatically — select all 3 new disks, choose **GPT (GUID Partition Table)** → **OK**.
7. Right-click unallocated space on Disk 1 → **New RAID-5 Volume… → Next**.
8. Select Disk 2 and Disk 3 → **Add >** to move them into the Selected column (3 disks total) → **Next**.
9. Assign Drive Letter: `E:` → **Next**.
10. **Format Volume:** Volume label: `DATA` · Perform a quick format: checked → **Next → Finish**.
11. Click **Yes** when warned that basic disks will be converted to dynamic disks. The volume initializes and formats as RAID 5 (`E:` drive with parity).

---

## 🖥️ Part 3: Deploy Windows 11 Pro (Domain Client)

### 3.1 VM Creation (OVMF / UEFI + TPM 2.0)

1. Proxmox UI → **Create VM**.
2. **General tab:** VM ID: auto · Name: `Win11Pro`
3. **OS tab:** **Use CD/DVD disc image file (ISO)** · Storage: `local` · ISO Image: Windows 11 ISO · Guest OS Type: **Microsoft Windows** · Version: **11/2022/2025**
4. **System tab:** Graphic card: Default · BIOS: **OVMF (UEFI)** · EFI Storage: `local-lvm` · Machine: `q35` (or default `i440fx`) · Pre-enroll keys: checked · Add TPM: checked (v2.0, Storage: `local-lvm`) · Qemu Agent: checked
5. **Disks tab:** Disk Size: **64 GB** · Storage: `local-lvm` · Bus/Device: **VirtIO SCSI** (default)
6. **CPU tab:** Cores: 2
7. **Memory tab:** Memory: **4096 MB**
8. **Network tab:** Bridge: `vmbr0` (temporary setup bridge)
9. Review and click **Finish** (don't start the VM yet).
10. Select `Win11Pro` → **Hardware → Add → CD/DVD Drive**. Storage: `local` · ISO Image: `virtio-win.iso`. This attaches the VirtIO drivers as a second, separate CD/DVD drive alongside the Windows 11 ISO.

### 3.2 OS Installation & Profile Configuration

1. Start VM `Win11Pro` → open **Console**.
2. Press any key immediately on boot to load the Windows Installer.
3. Select language/keyboard preferences → **Next → Install now**.
4. When prompted for a product key → **I don't have a product key**.
5. Select Edition: **Windows 11 Pro** → **Next**.
6. Accept the EULA → **Custom: Install Windows only (advanced)**.
7. On the drive selection screen, **no drives will be listed** — this is expected with a VirtIO SCSI controller. Click **Load driver**.
8. Uncheck "Hide drivers that aren't compatible with this computer's hardware," then click **Browse**.
9. Navigate to the VirtIO CD drive → `vioscsi` → `w11` → `amd64`, then click **OK**.
10. Select the listed **Red Hat VirtIO SCSI controller** driver → **Next** to load it. The 64 GB virtual disk should now appear.
11. Select the 64 GB partition → **Next**.
12. Complete the Out-of-Box Experience (OOBE):
   - Local Account Name: `Student`
   - Password: `Password1`
8. Allow the Windows 11 desktop to load completely.

### 3.3 Network Attachment (`vmbr1`)

1. Proxmox UI → VM `Win11Pro` → **Hardware** → double-click **Network Device** (`net0`).
2. Set Bridge to `vmbr1` (Internal Isolated Network) → **OK**.

### 3.4 Join Active Directory Domain

1. Boot the Windows 11 VM and log in as local `Student`.
2. Open Command Prompt (`cmd`) and run:

```dos
ipconfig /all
```

Verify the network interface received an IP address in the `192.168.0.11`–`.254` range, with Gateway blank and DNS set to `192.168.0.10`.

3. Test network connectivity to the DC:

```dos
ping dc1.yourname.local
```

4. **Join Domain:**
   - Press **Win + R** → type `sysdm.cpl` → **Enter** (System Properties).
   - Under the **Computer Name** tab → **Change…**.
   - Member of: select **Domain** → type `yourname.local` → **OK**.
   - When prompted for domain credentials:
     - Username: `yourname.local\Administrator`
     - Password: `Password1`
   - A dialog displays "Welcome to the yourname.local domain." → **OK**.
5. Restart the computer when prompted.

### 3.5 Domain Logon Verification

1. On the Windows 11 logon screen, click **Other user** in the lower-left corner.
2. Enter domain credentials for one of the AD users created in step 2.6:
   - User: `yourname.local\user1` (or `user1@yourname.local`)
   - Password: `Password1`
3. Windows 11 initializes the new user profile.
4. Verify domain identity in CMD:

```dos
whoami
```

Output should show: `yourname\user1`

### 3.6 Storage Subsystem: Software RAID 0 (Mirrored Volume)

Demonstrates client-level drive mirroring (RAID 1) using dynamic disks:

1. Shut down VM `Win11Pro`.
2. Proxmox UI → VM `Win11Pro` → **Hardware → Add → Hard Disk**. Storage: `local-lvm` · Disk Size: `5 GB` → **Add**.
3. Repeat once more so two 5 GB virtual disks are attached.
4. Power on `Win11Pro` and log in as `yourname\Administrator` (or local admin).
5. Open **Computer Management** (`compmgmt.msc`) → **Disk Management**.
6. The Initialize Disks dialog prompts — ensure Disk 1 and Disk 2 are selected, choose **GPT** → **OK**.
7. Right-click unallocated space on Disk 1 → **New Mirrored Volume… → Next**.
8. Select Disk 2 under available disks → **Add >** to move it to Selected disks → **Next**.
9. Assign Drive Letter: `F:` → **Next**.
10. Volume Label: `DATA` · select **Perform a quick format** → **Next → Finish**.
11. Accept the dynamic disk conversion prompt (**Yes**). Storage volume `F:` is now mirrored across both 5 GB virtual disks.

---

## 🔍 Technical Reference & Best Practices

> **Tip — Thin Provisioning vs Pre-allocation**
> In Proxmox VE, storage pools configured with Thin Provisioning (e.g., `local-lvm`) allow guest virtual disks to consume storage on-demand rather than allocating full provisioned sizes upfront. Keep Thin Provisioning enabled for lab environments to maximize host storage efficiency.

> **Tip — VirtIO Drivers**
> Proxmox's default **VirtIO SCSI** disk controller offers much better I/O performance than emulated SATA/IDE, but Windows Setup can't see the disk without the driver loaded from `virtio-win.iso` during installation. After the OS is installed, you can leave the VirtIO CD/DVD drive attached and run the installer inside it (`virtio-win-guest-tools.exe`) to install the rest of the VirtIO drivers (network, balloon, QEMU Guest Agent) for full performance and the ability to see the VM's IP/stats in the Proxmox UI.

> **Note — Virtualization Flags**
> Ensure hardware virtualization extensions (Intel VT-x / AMD-V) are exposed to guest OS instances if nested virtualization is required. In Proxmox, CPU type can be set to `host` under VM → Hardware → Processor for optimal performance.

> **Important — DHCP DORA Handshake**
> The active IP assignment on `Win11Pro` follows the standard 4-step DHCP protocol across `vmbr1`:
> 1. **Discover:** Client broadcasts request for a DHCP server.
> 2. **Offer:** `DC1` offers an available IP from `Scope1` (`192.168.0.11`).
> 3. **Request:** Client accepts the offered IP configuration.
> 4. **Acknowledge:** `DC1` commits the lease and confirms binding.

> **Tip — Group Policy Management (GPO)**
> Expand management capabilities by opening Group Policy Management (`gpmc.msc`) on `DC1`. Link organizational unit policies (e.g., mapping drive `E:` or `F:` automatically, password policies, or Windows Firewall configurations) directly to the `Departments`, `Sales`, `HR`, and `Marketing` OUs.

## 📜 License

This project documentation and configuration scripts are open-source and available under the MIT License. Hypervisor software (Proxmox VE) is licensed under GNU AGPLv3. Microsoft Windows Server and Windows 11 are subject to Microsoft software licensing terms.
