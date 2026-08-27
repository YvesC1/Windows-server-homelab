# 🚀 Enterprise Virtual Machine Lab: Proxmox VE, Windows Server 2022 & Windows 11 Pro

[![Proxmox VE](https://img.shields.io/badge/Hypervisor-Proxmox%20VE%208.x-E65100?style=for-the-badge&logo=proxmox&logoColor=white)](https://www.proxmox.com/)
[![Windows Server](https://img.shields.io/badge/Domain%20Controller-Windows%20Server%202022-0078D4?style=for-the-badge&logo=windows-server&logoColor=white)](https://www.microsoft.com/evalcenter/evaluate-windows-server-2022)
[![Windows 11](https://img.shields.io/badge/Client%20OS-Windows%2011%20Pro-0078D4?style=for-the-badge&logo=windows11&logoColor=white)](https://www.microsoft.com/software-download/windows11)
[![License](https://img.shields.io/badge/License-MIT-green.svg?style=for-the-badge)](LICENSE)
[![Cost](https://img.shields.io/badge/Cost-$0%20(Open%20Source%20%2B%20Eval)-brightgreen?style=for-the-badge)](#-cost--licensing)

A comprehensive, step-by-step engineering guide for building an isolated, production-grade Active Directory Domain Services (AD DS) lab environment. This project covers setting up a bare-metal **Proxmox VE** hypervisor, deploying a **Windows Server 2022** Domain Controller with DHCP and software RAID 5, and configuring a **Windows 11 Pro** client VM joined to the domain with mirrored storage.

---

## 📋 Table of Contents

- [ Architecture & Topology](#-architecture--topology)
- [ Environment Specifications](#-environment-specifications)
- [ Cost & Licensing](#-cost--licensing)
- [ Part 1: Set Up Proxmox VE Hypervisor](#part-1-set-up-proxmox-ve-hypervisor)
  - [1.1 Download & Verify ISOs](#11-download--verify-isos)
  - [1.2 Create Bootable USB Media](#12-create-bootable-usb-media)
  - [1.3 Bare-Metal Installation](#13-bare-metal-installation)
  - [1.4 Post-Install: Free Repository Setup](#14-post-install-free-repository-setup)
  - [1.5 Upload ISOs to Storage](#15-upload-isos-to-storage)
- [ Part 2: Deploy Windows Server 2022 (Domain Controller)](#part-2-deploy-windows-server-2022-domain-controller)
  - [2.1 VM Creation (SeaBIOS)](#21-vm-creation-seabios)
  - [2.2 OS Installation & Initial Credentials](#22-os-installation--initial-credentials)
  - [2.3 Network Isolation (`vmbr1`)](#23-network-isolation-vmbr1)
  - [2.4 Host Name, Time Zone & Static IP Setup](#24-host-name-time-zone--static-ip-setup)
  - [2.5 Install & Configure AD DS (Forest Setup)](#25-install--configure-ad-ds-forest-setup)
  - [2.6 Structure Organizational Units & Users](#26-structure-organizational-units--users)
  - [2.7 Deploy & Authorize DHCP Role](#27-deploy--authorize-dhcp-role)
  - [2.8 Configure DHCP Scope & Exclusions](#28-configure-dhcp-scope--exclusions)
  - [2.9 Storage Subsystem: Software RAID 5 Setup](#29-storage-subsystem-software-raid-5-setup)
- [ Part 3: Deploy Windows 11 Pro (Domain Client)](#part-3-deploy-windows-11-pro-domain-client)
  - [3.1 VM Creation (OVMF / UEFI + TPM 2.0)](#31-vm-creation-ovmf--uefi--tpm-20)
  - [3.2 OS Installation & Profile Configuration](#32-os-installation--profile-configuration)
  - [3.3 Network Attachment (`vmbr1`)](#33-network-attachment-vmbr1)
  - [3.4 Join Active Directory Domain](#34-join-active-directory-domain)
  - [3.5 Domain Logon Verification](#35-domain-logon-verification)
  - [3.6 Storage Subsystem: Software RAID 0 (Mirrored Volume)](#36-storage-subsystem-software-raid-0-mirrored-volume)
- [ Technical Reference & Best Practices](#-technical-reference--best-practices)
- [ License](#-license)

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
   | - Storage: System Disk + RAID 5 (3x 5GB)    |             | - Storage: System Disk + Mirrored (2x 5GB)  |
   +---------------------------------------------+             +---------------------------------------------+
```

---

## 📊 Environment Specifications

| Parameter | Proxmox VE Host | Windows Server 2022 (DC1) | Windows 11 Pro Client |
| :--- | :--- | :--- | :--- |
| **Role** | Bare-Metal Hypervisor | Domain Controller / DHCP / DNS | Enterprise Client Workstation |
| **BIOS Mode** | UEFI / Legacy BIOS | SeaBIOS (Default) | OVMF (UEFI) + TPM 2.0 |
| **vCPU Cores** | Host Hardware (Min 2 Cores) | 2 vCPU | 2 vCPU |
| **RAM** | 8 GB – 16 GB+ | 2048 MB | 4096 MB |
| **Primary Storage** | 32 GB+ (ext4/ZFS) | 50 GB (Thin Provisioned) | 64 GB (Thin Provisioned) |
| **Data Storage** | - | 3 x 5 GB Virtual Disks (RAID 5) | 2 x 5 GB Virtual Disks (RAID 1 Mirror) |
| **Network Interface**| `vmbr0` (Physical Management) | `vmbr1` (Internal Isolated) | `vmbr1` (Internal Isolated) |
| **IP Allocation** | Static Management IP | Static (`192.168.0.10/24`) | Dynamic (`192.168.0.11`–`.254`) |

---

## 💰 Cost & Licensing

> [!NOTE]
> Proxmox VE is free and open-source software under the **GNU AGPLv3** license. This complete lab infrastructure can be deployed at **$0 licensing cost**.
> 
> - **Proxmox VE:** Fully functional without subscription. Access to the `pve-no-subscription` repository is free forever.
> - **Windows Server 2022 & Windows 11 Pro:** Deployed using Microsoft Evaluation ISOs, providing full feature functionality for testing and training.
> - **Hardware Efficiency:** Ideal for non-profits, homelabs, or corporate training labs looking to consolidate server hardware without per-seat hypervisor licensing fees.

---

## 🛠️ Part 1: Set Up Proxmox VE Hypervisor

### 1.1 Download & Verify ISOs

1. Download the latest official Proxmox VE installation ISO from the [Proxmox Downloads Page](https://www.proxmox.com/en/downloads).
2. Verify the downloaded ISO checksum using PowerShell (Windows) or terminal (Linux/macOS):
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
- **Rufus (Windows):** Select target USB -> Partition scheme: **GPT** -> Target system: **UEFI**.
  
  > [!IMPORTANT]
  > When prompted by Rufus, select **Write in DD Image mode**. Proxmox uses a hybrid ISO structure that requires raw sector writing.

- **`dd` (Linux/macOS):**
  ```bash
  sudo dd if=proxmox-ve.iso of=/dev/sdX bs=1M status=progress conv=fsync
  ```

### 1.3 Bare-Metal Installation

1. Insert the bootable USB into the target machine and enter system BIOS/UEFI. Verify that hardware virtualization extensions (**Intel VT-x** or **AMD-V**) are enabled.
2. Boot from the USB stick and select **Install Proxmox VE (Graphical)**.
3. Accept the End User License Agreement (EULA).
4. **Target Harddisk:** Select target install disk. Click **Options** to select the filesystem:
   - `ext4` (LVM): Recommended default for single-drive setups.
   - `ZFS (RAID1/10/39)`: Recommended only if multi-drive redundancy and extra system RAM (8 GB+) are available.
5. **Location and Timezone:** Set country, timezone, and keyboard layout.
6. **Password and Email:** Specify root administration password and email notification address.
7. **Management Network Configuration:**
   - **Management Interface:** Select physical NIC.
   - **Hostname (FQDN):** Set full domain name (e.g., `pve.lab.local`). *Do not use `.local` as root domain if mDNS conflicts are expected.*
   - **IP Address / Netmask / Gateway / DNS:** Set static management IP parameters.
8. Complete installation and reboot. Log into the web administration UI at `https://<your-proxmox-ip>:8006`.

### 1.4 Post-Install: Free Repository Setup

To update Proxmox VE without an enterprise subscription:

1. Navigate to **Node** -> **Repositories**.
2. Disable the `pve-enterprise` repository.
3. Click **Add** -> Select **No-Subscription** repository (`pve-no-subscription`).
4. Go to **APT** -> Click **Refresh** -> Click **Upgrade** to pull the latest system packages.

### 1.5 Upload ISOs to Storage

1. Download ISO files for:
   - **Windows Server 2022 Evaluation** (Microsoft Evaluation Center)
   - **Windows 11 Pro** (Microsoft Windows 11 Download Page)
2. In Proxmox Web UI: Navigate to **Server Node** -> **local (pve)** -> **ISO Images**.
3. Click **Upload** and upload both Windows Server 2022 and Windows 11 ISOs.

---

## 💻 Part 2: Deploy Windows Server 2022 (Domain Controller)

### 2.1 VM Creation (SeaBIOS)

1. Click **Create VM** in the top right header.
2. **General Tab:**
   - VM ID: Auto-assigned or custom (e.g., `100`)
   - Name: `WinSer22`
3. **OS Tab:**
   - Media: Select **Use CD/DVD disc image file (ISO)**
   - Storage: `local` | ISO Image: `Windows Server 2022 ISO`
   - Type: `Microsoft Windows` | Version: `11/2022/2025`
4. **System Tab:**
   - Graphic card: `Default`
   - BIOS: `Default (SeaBIOS)`
   - Machine: `Default (i440fx)`
5. **Disks Tab:**
   - Storage: `local-lvm`
   - Disk size: `50 GB`
   - Format: `Raw disk image` (or `qcow2`)
6. **CPU Tab:**
   - Cores: `2` (minimum recommended)
7. **Memory Tab:**
   - Memory: `2048 MB`
8. **Network Tab:**
   - Bridge: `vmbr0` *(Temporary for setup; isolated in step 2.3)*
9. Review **Confirm** summary and click **Finish**.

### 2.2 OS Installation & Initial Credentials

1. Select VM **WinSer22** -> Click **Start** -> Click **Console (noVNC)**.
2. Press any key when prompted to boot from CD/DVD.
3. Setup Configuration:
   - Language / Time Format: Choose preferred options -> Click **Next** -> **Install Now**.
   - Edition: Select **Windows Server 2022 Standard Evaluation (Desktop Experience)**.
   - Accept license terms -> Select **Custom: Install Windows only (advanced)**.
   - Select unallocated space (50 GB) -> Click **Next**.
4. Upon reboot, set the built-in Administrator password:
   - **Password:** `Password1`
5. Click **Finish** and log in via Console (`Ctrl+Alt+Del` button at top-left of noVNC).

> [!TIP]
> Always shut down Windows guests cleanly from inside the OS (`Start` -> `Power` -> `Shut down`) rather than using Proxmox's hard **Stop** button to avoid filesystem corruption.

### 2.3 Network Isolation (`vmbr1`)

To prevent lab traffic (DHCP, AD broadcast) from polluting the host/production network:

1. In Proxmox UI, navigate to **Datacenter** -> **Node** -> **System** -> **Network**.
2. Click **Create** -> **Linux Bridge**.
   - Name: `vmbr1`
   - IPv4/CIDR: *Leave Blank*
   - Gateway: *Leave Blank*
   - Bridge ports: *Leave Blank (No physical NIC attached)*
   - Comment: `Internal Lab Network`
3. Click **Create** -> Apply Configuration.
4. Go to VM **WinSer22** -> **Hardware** -> Double-click **Network Device (net0)**.
5. Change Bridge to **`vmbr1`** -> Click **OK**.

### 2.4 Host Name, Time Zone & Static IP Setup

#### Set Time Zone
1. Open **Server Manager** -> Select **Local Server**.
2. Click current **Time zone** link -> Click **Change time zone...**.
3. Select `(UTC-05:00) Eastern Time (US & Canada)` -> Click **OK**.

#### Configure Static Network Interface
1. In Server Manager -> **Local Server** -> Click network adapter link next to **IPv4 address**.
2. Right-click network interface -> **Properties**.
3. Uncheck **Internet Protocol Version 6 (TCP/IPv6)**.
4. Select **Internet Protocol Version 4 (TCP/IPv4)** -> Click **Properties**.
5. Configure static parameters:
   - **IP address:** `192.168.0.10`
   - **Subnet mask:** `255.255.255.0`
   - **Default gateway:** *Leave Blank*
   - **Preferred DNS server:** `192.168.0.10` *(Self-referencing for AD DS)*
6. Click **OK** -> **Close**.

#### Rename Computer Hostname
1. Server Manager -> **Local Server** -> Click **Computer name** link.
2. In System Properties -> Click **Change...**.
3. Computer Name: `DC1` -> Click **OK**.
4. Restart the virtual machine when prompted.

---

### 2.5 Install & Configure AD DS (Forest Setup)

1. Log back into `DC1` as Administrator (`Password1`).
2. Server Manager opens automatically -> Click **Manage** -> **Add Roles and Features**.
3. **Installation Type:** Select **Role-based or feature-based installation** -> **Next**.
4. **Server Selection:** Ensure `DC1` is selected -> **Next**.
5. **Server Roles:** Check **Active Directory Domain Services**.
   - When dialog pops up, click **Add Features** -> Click **Next**.
6. **Features & AD DS Pages:** Click **Next** through defaults.
7. **Confirmation:** Check *"Restart the destination server automatically if required"* -> Click **Install**.

#### Promote Server to Domain Controller
1. Once installation completes, click the **Notification Flag** (top right) -> Click **Promote this server to a domain controller**.
2. **Deployment Configuration:**
   - Select **Add a new forest**
   - Root domain name: `yourname.local` -> Click **Next**
3. **Domain Controller Options:**
   - Domain Functional Level: `Windows Server 2016`
   - Password / DSRM Password: `Password1` -> Click **Next**
4. **DNS Options:** Click **Next** *(ignore delegation warning)*.
5. **Additional Options:** Verify NetBIOS domain name (`YOURNAME`) -> Click **Next**.
6. **Paths / Review:** Leave default database/log/SYSVOL paths -> Click **Next**.
7. **Prerequisites Check:** Ensure checks pass -> Click **Install**.
8. Server will automatically restart upon completion. Log back in as `YOURNAME\Administrator` with password `Password1`.

---

### 2.6 Structure Organizational Units & Users

1. Open **Server Manager** -> **Tools** -> **Active Directory Users and Computers**.
2. Expand domain node `yourname.local`.
3. Create Main Organizational Unit (OU):
   - Right-click `yourname.local` -> **New** -> **Organizational Unit**.
   - Name: `Departments` -> Click **OK**.
4. Create Sub-OUs:
   - Right-click `Departments` -> **New** -> **Organizational Unit** -> Name: `Sales`
   - Right-click `Departments` -> **New** -> **Organizational Unit** -> Name: `HR`
   - Right-click `Departments` -> **New** -> **Organizational Unit** -> Name: `Marketing`
5. Provision User Accounts:
   - Right-click **Sales** OU -> **New** -> **User**:
     - First name: `User1` | User logon name: `user1` -> Click **Next**.
     - Password: `Password1`
     - **Uncheck** *"User must change password at next logon"*
     - **Check** *"Password never expires"* -> Click **Next** -> **Finish**.
   - Repeat process for **User2** inside **HR** OU and **User3** inside **Marketing** OU.

---

### 2.7 Deploy & Authorize DHCP Role

1. Server Manager -> **Manage** -> **Add Roles and Features** -> **Next**.
2. Select **Role-based or feature-based installation** -> Select `DC1`.
3. **Server Roles:** Check **DHCP Server** -> Click **Add Features** -> Click **Next**.
4. Click **Next** through Features -> Click **Install**.
5. Post-Deployment Configuration:
   - Click **Notification Flag** -> Click **Complete DHCP configuration**.
   - Click **Next** on Description page.
   - Authorization: Select **Use the target computer's credentials** (`YOURNAME\Administrator`) -> Click **Commit**.
   - Click **Close**.

---

### 2.8 Configure DHCP Scope & Exclusions

1. Open Server Manager -> **Tools** -> **DHCP**.
2. Expand `dc1.yourname.local` -> Right-click **IPv4**.
   
   > [!NOTE]
   > If IPv4 icon exhibits a red status icon, right-click **IPv4** -> Click **Authorize**, then click **Refresh**.

3. Right-click **IPv4** -> Select **New Scope...** -> Click **Next**.
4. **Scope Name:** `Scope1` -> **Next**.
5. **IP Address Range:**
   - Start IP Address: `192.168.0.1`
   - End IP Address: `192.168.0.254`
   - Length: `24` | Subnet Mask: `255.255.255.0`
   - Click **Next**.
6. **Add Exclusions and Delay:**
   - Start IP: `192.168.0.1` | End IP: `192.168.0.10`
   - Click **Add** -> Click **Next**.
7. **Lease Duration:** Keep default (8 days) -> **Next**.
8. **Configure DHCP Options:** Select *Yes, I want to configure these options now* -> **Next**.
9. **Router (Default Gateway):** Click **Next** (Leave blank for isolated network).
10. **Domain Name and DNS Servers:** Confirm parent domain is `yourname.local` and IP `192.168.0.10` is listed -> Click **Next**.
11. **WINS Servers:** Click **Next**.
12. **Activate Scope:** Select *Yes, I want to activate this scope now* -> Click **Next** -> **Finish**.

---

### 2.9 Storage Subsystem: Software RAID 5 Setup

Demonstrating Windows Server software storage redundancy using dynamic disks:

1. In Proxmox UI, shut down VM **WinSer22**.
2. Select **WinSer22** -> **Hardware** -> Click **Add** -> **Hard Disk**.
   - Storage: `local-lvm` | Disk Size: `5 GB` -> Click **Add**.
3. Repeat step 2 two additional times so **three extra 5 GB virtual disks** are attached (`scsi1`, `scsi2`, `scsi3`).
4. Boot VM **WinSer22** and log in as Administrator.
5. Right-click Start Menu -> Select **Disk Management**.
6. **Initialize Disk** dialog will prompt automatically:
   - Select all 3 new disks (Disk 1, Disk 2, Disk 3).
   - Partition style: Choose **GPT (GUID Partition Table)** -> Click **OK**.
7. Right-click unallocated space on Disk 1 -> Select **New RAID-5 Volume...** -> Click **Next**.
8. Select Disk 2 and Disk 3 from left column -> Click **Add >** to move them into *Selected* column (Total 3 disks) -> Click **Next**.
9. Assign Drive Letter: `E:` -> Click **Next**.
10. Format Volume:
    - Volume label: `DATA`
    - Perform a quick format: **Checked**
    - Click **Next** -> **Finish**.
11. Click **Yes** when warned that basic disks will be converted to dynamic disks. Volume will initialize and format as RAID 5 (`E:` drive with parity).

---

## 🖥️ Part 3: Deploy Windows 11 Pro (Domain Client)

### 3.1 VM Creation (OVMF / UEFI + TPM 2.0)

1. Proxmox UI -> Click **Create VM**.
2. **General Tab:** VM ID: Auto | Name: `Win11Pro`
3. **OS Tab:**
   - Select **Use CD/DVD disc image file (ISO)**
   - Storage: `local` | ISO Image: `Windows 11 ISO`
   - Guest OS Type: `Microsoft Windows` | Version: `11/2022/2025`
4. **System Tab:**
   - Graphic card: `Default`
   - BIOS: **`OVMF (UEFI)`**
   - EFI Storage: `local-lvm`
   - Machine: `q35` (or default `i440fx`)
   - Pre-enroll keys: Checked
   - **Add TPM:** Checked (`v2.0` | Storage: `local-lvm`)
   - **Qemu Agent:** Checked
5. **Disks Tab:** Disk Size: `64 GB` | Storage: `local-lvm`
6. **CPU Tab:** Cores: `2`
7. **Memory Tab:** Memory: `4096 MB`
8. **Network Tab:** Bridge: `vmbr0` (Temporary setup bridge)
9. Review and click **Finish**.

---

### 3.2 OS Installation & Profile Configuration

1. Start VM **Win11Pro** -> Open **Console**.
2. Press any key immediately on boot to load Windows Installer.
3. Select language/keyboard preferences -> Click **Next** -> **Install now**.
4. When prompted for Product Key -> Click **I don't have a product key**.
5. Select Edition: **Windows 11 Pro** -> Click **Next**.
6. Accept EULA -> Select **Custom: Install Windows only (advanced)** -> Select 64 GB partition -> Click **Next**.
7. Complete Out-of-Box Experience (OOBE):
   - Local Account Name: `Student`
   - Password: `Password1`
8. Allow Windows 11 desktop to load completely.

---

### 3.3 Network Attachment (`vmbr1`)

1. Shut down or modify network interface live:
   - Proxmox UI -> VM **Win11Pro** -> **Hardware** -> Double-click **Network Device (net0)**.
   - Set Bridge to **`vmbr1`** (Internal Isolated Network) -> Click **OK**.

---

### 3.4 Join Active Directory Domain

1. Boot Windows 11 VM and log in as local `Student`.
2. Open **Command Prompt** (`cmd`) and run:
   ```cmd
   ipconfig /all
   ```
   *Verify that the network interface received an IP address in the `192.168.0.11`–`.254` range, with Gateway blank and DNS set to `192.168.0.10`.*

3. Test network Connectivity to DC:
   ```cmd
   ping dc1.yourname.local
   ```
4. Join Domain:
   - Press `Win + R` -> type `sysdm.cpl` -> press **Enter** (System Properties).
   - Under **Computer Name** tab -> Click **Change...**.
   - Member of: Select **Domain** -> Type: `yourname.local` -> Click **OK**.
5. When prompted for Domain Credentials:
   - **Username:** `yourname.local\Administrator`
   - **Password:** `Password1`
6. A dialog box will display: *"Welcome to the yourname.local domain."* -> Click **OK**.
7. Restart the computer when prompted.

---

### 3.5 Domain Logon Verification

1. On the Windows 11 logon screen, click **Other user** in the lower-left corner.
2. Enter domain credentials for one of the AD users created in step 2.6:
   - **User:** `yourname.local\user1` (or `user1@yourname.local`)
   - **Password:** `Password1`
3. Windows 11 will initialize the new user profile.
4. Verify domain identity in CMD:
   ```cmd
   whoami
   ```
   *Output should show:* `yourname\user1`

---

### 3.6 Storage Subsystem: Software RAID 0 (Mirrored Volume)

Demonstrating client-level drive mirroring (RAID 1) using dynamic disks:

1. Shut down VM **Win11Pro**.
2. Proxmox UI -> VM **Win11Pro** -> **Hardware** -> Click **Add** -> **Hard Disk**.
   - Storage: `local-lvm` | Disk Size: `5 GB` -> Click **Add**.
3. Repeat step 2 so **two 5 GB virtual disks** are attached.
4. Power on VM **Win11Pro** and log in as `yourname\Administrator` (or local admin).
5. Open **Computer Management** (`compmgmt.msc`) -> Select **Disk Management**.
6. Initialize Disks dialog will prompt -> Ensure Disk 1 and Disk 2 are selected -> Choose **GPT** -> Click **OK**.
7. Right-click unallocated space on Disk 1 -> Select **New Mirrored Volume...** -> Click **Next**.
8. Select **Disk 2** under available disks -> Click **Add >** to add it to Selected disks -> Click **Next**.
9. Assign Drive Letter: `F:` -> Click **Next**.
10. Volume Label: `DATA` -> Select **Perform a quick format** -> Click **Next** -> **Finish**.
11. Accept dynamic disk conversion prompt (**Yes**). Storage volume `F:` is now mirrored across both 5 GB virtual disks.

---

## 🔍 Technical Reference & Best Practices

> [!TIP]
> ### Thin Provisioning vs Pre-allocation
> In Proxmox VE, storage pools configured with **Thin Provisioning** (e.g., `local-lvm`) allow guest virtual disks to consume storage on-demand rather than allocating full provisioned sizes upfront. Keep Thin Provisioning enabled for lab environments to maximize host storage efficiency.

> [!NOTE]
> ### Virtualization Flags
> Ensure hardware virtualization extensions (**Intel VT-x** / **AMD-V**) are exposed to guest OS instances if nested virtualization is required. In Proxmox, CPU type can be set to **`host`** under **VM** -> **Hardware** -> **Processor** for optimal performance.

> [!IMPORTANT]
> ### DHCP DORA Handshake
> The active IP assignment on `Win11Pro` follows the standard 4-step DHCP protocol across `vmbr1`:
> 1. **Discover:** Client broadcasts request for DHCP server.
> 2. **Offer:** `DC1` offers available IP from `Scope1` (`192.168.0.11`).
> 3. **Request:** Client accepts offered IP configuration.
> 4. **Acknowledge:** `DC1` commits lease and confirms binding.

> [!TIP]
> ### Group Policy Management (GPO)
> Expand management capabilities by opening **Group Policy Management** (`gpmc.msc`) on `DC1`. Link organizational unit policies (e.g., mapping drive `E:` or `F:` automatically, password policies, or Windows Firewall configurations) directly to the `Departments`, `Sales`, `HR`, and `Marketing` OUs.

---

## 📜 License

This project documentation and configuration scripts are open-source and available under the [MIT License](LICENSE). Hypervisor software (Proxmox VE) is licensed under GNU AGPLv3. Microsoft Windows Server and Windows 11 are subject to Microsoft software licensing terms.
