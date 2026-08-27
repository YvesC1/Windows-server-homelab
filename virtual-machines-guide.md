# How to Deploy Virtual Machines: Proxmox, Windows Server & Windows 11 Pro

A step-by-step guide for standing up a small AD DS lab: a Proxmox VE hypervisor, a Windows Server 2022 domain controller, and a Windows 11 Pro client joined to the domain.

## Table of Contents

1. [Part 1: Set Up Proxmox VE](#part-1-set-up-proxmox-ve)
   - [Install Proxmox VE](#install-proxmox-ve)
   - [Download the required ISOs](#download-the-required-isos)
2. [Part 2: Create the Windows Server 2022 Virtual Machine](#part-2-create-the-windows-server-2022-virtual-machine)
3. [Part 3: Create the Windows 11 Pro Virtual Machine](#part-3-create-the-windows-11-pro-virtual-machine)
4. [Notes](#notes)

> 💡 **Cost note:** Proxmox VE is free and open source (GPL v2), so this whole lab can be built at zero licensing cost. The no-subscription repo is free forever — the only paid tier is optional enterprise support.

---

## Part 1: Set Up Proxmox VE

Proxmox runs as a bare-metal host, so you'll want a dedicated box (or an old PC/mini PC) rather than an app installed on your daily-driver machine — this is what lets multiple people access the lab remotely, which is useful for a nonprofit consolidating a few servers onto one cheap piece of hardware instead of buying per-seat virtualization licensing.

### Install Proxmox VE

1. **Download the ISO.** Grab the latest Proxmox VE ISO from the official Proxmox downloads page, and verify the published SHA256 checksum against your download before using it.
2. **Create bootable media.** Flash the ISO to an 8 GB+ USB stick with Rufus (Windows) or `dd` (Linux/macOS). In Rufus, choose GPT partitioning and **DD Image mode** — Proxmox's hybrid ISO only writes correctly as a raw image.
3. **Boot the installer.** Plug the USB into the target machine, boot from it, and choose **Install Proxmox VE (Graphical)**.
4. **Accept the EULA**, then pick your target disk. Click **Options** to choose a filesystem — `ext4` (LVM) is the simple default for a single disk; ZFS is worth it only if you have 2+ disks and enough RAM to spare (ZFS wants extra memory for caching).
5. Set your **location, time zone, and root password**, then set networking — give the host a proper FQDN (e.g. `pve.lab.local`) rather than a bare hostname, and avoid the `.local` suffix since it conflicts with mDNS.
6. Finish the install and reboot. Log into the Proxmox web UI at `https://<server-ip>:8006`.
7. **Switch to the free/no-subscription repository** so the host can update without an enterprise license — this is what lets you run Proxmox at zero cost indefinitely.

> Minimum requirements: 64-bit CPU with virtualization support (Intel VT-x/AMD-V), 2 GB RAM technically, but 8 GB+ realistic if using ZFS, and 32 GB storage. For this lab (a DC + a client VM) budget more like 8–16 GB RAM total.

### Download the required ISOs

1. **Windows 11 ISO:** Go to the Microsoft Windows 11 download page, click **Download Now** under "Download Windows 11 Disk Image (ISO) for x64 devices," and choose your language.
2. **Windows Server 2022 ISO:** Go to the Microsoft Evaluation Center and click **Download Now**.
3. In the Proxmox web UI, go to your node → **local (storage)** → **ISO Images** → **Upload**, and upload both ISO files so they're available when creating VMs.

---

## Part 2: Create the Windows Server 2022 Virtual Machine

### Create and launch the VM

1. In the Proxmox web UI, click **Create VM** (top right).
2. **General** tab: give it a VM ID and set Name to **WinSer22**.
3. **OS** tab: select **Use CD/DVD disc image file (ISO)**, choose the Windows Server ISO from storage, and set Guest OS Type to **Microsoft Windows** / version **11/2022/2025**.
4. **System** tab: set BIOS to **Default (SeaBIOS)** and leave Machine as default (`i440fx`) for this lab.
5. **Disks** tab: set the disk size to **50 GB** minimum, leave the default storage/bus.
6. **CPU** tab: leave defaults (2 cores is fine).
7. **Memory** tab: set to **2048 MB**.
8. **Network** tab: leave the bridge as `vmbr0` for now (you'll isolate it in the next step).
9. **Confirm** tab: review the summary and click **Finish**.
10. Select the VM in the left pane → **Start**, then open its **Console** (noVNC) to watch the install.
11. In Windows Server Setup: **Next** → **Install now** → select **Windows Server 2022 Standard Evaluation (Desktop Experience)** → **Next** → accept the license terms → **Custom** install → **Next** to begin installing.
12. Once installed, the VM restarts and prompts you to set a password (use `Password1`) — click **Finish**.
13. Log in with `Password1` from the console.
14. You're now logged into Windows Server 2022 (Server Manager opens automatically by default).
15. To power off cleanly, shut down from inside Windows (Start → Power → Shut down) rather than using Proxmox's **Stop** button.

### Isolate the network adapter

1. Select the VM → **Hardware** → double-click **Network Device**.
2. Set the bridge to an internal-only bridge (e.g. create `vmbr1` with no physical NIC attached under **Datacenter → node → System → Network**, or use a Proxmox SDN "internal" zone), so this lab traffic stays isolated from your production network. Click **OK**.

### Configure the time zone

1. Start the VM, log in from the console.
2. In Server Manager, click **Local Server** in the left pane.
3. Click the hyperlink next to **Time zone**.
4. Click **Change time zone…**, set it to **(UTC-05:00) Eastern Time (US & Canada)**, click **OK** twice.
5. Restart the server to apply.

### Set a static IP

1. Log in and let Server Manager open.
2. Click **Local Server** → click the hyperlink for the IPv4 address assigned by DHCP.
3. Right-click the Ethernet adapter → **Properties**.
4. Uncheck IPv6, double-click IPv4.
5. Select **Use the following IP address**: IP `192.168.0.10`, subnet mask `255.255.255.0`, leave the default gateway blank, and set the preferred DNS server to `192.168.0.10`. Click **OK** twice to save.

### Rename the computer

1. In Server Manager, click the **Computer name** hyperlink to open System Properties.
2. Click **Change**.
3. Set the name to **DC1**, click **OK** twice — the VM restarts. Log back in with `Password1`.

### Install Active Directory Domain Services (AD DS)

1. In Server Manager, click **Manage** → **Add Roles and Features**.
2. **Before You Begin** → Next.
3. **Installation Type** → leave as Role-based/feature-based → Next.
4. **Server Selection** → leave as-is → Next.
5. **Server Roles** → check **Active Directory Domain Services** → **Add Features** when prompted → Next.
6. **Features** → Next.
7. **AD DS** page → Next.
8. **Confirmation** → check "Restart the destination server automatically if required" → **Install**.
9. When prompted, click **Yes** to allow restart, then **Install**.
10. Click the notification flag → **Promote this server to a domain controller**.
11. **Deployment Configuration** → **Add a new forest** → name the root domain e.g. `yourname.local` → Next.
12. **Domain Controller Options** → set a DSRM password (`Password1`) → Next.
13. **DNS Options** → Next.
14. **Additional Options** → NetBIOS name auto-populates → Next.
15. **Paths** → note the database/log/SYSVOL locations → Next.
16. **Review Options** → Next.
17. **Prerequisites Check** → confirm all checks pass → **Install**. The VM restarts automatically.
18. Log back in with `Password1`.

### Add Organizational Units and users

1. Open Server Manager → **Tools** → **Active Directory Users and Computers**.
2. Right-click your domain (e.g. `yourname.local`) → **New** → **Organizational Unit**, name it `Departments`.
3. Right-click **Departments** → **New** → **Organizational Unit** three times to create **Sales**, **HR**, and **Marketing**.
4. Right-click **Sales** → **New** → **User**. First name `User1`, logon name `user1` → Next.
5. Password `Password1`, uncheck "User must change password at next login" → Next → **Finish**.
6. Repeat for **User2** in HR and **User3** in Marketing with the same password settings.

### Configure DHCP

1. Server Manager → **Add Roles and Features**.
2. **Before You Begin** → Next.
3. **Installation Type** → Next.
4. **Server Selection** → Next.
5. **Server Roles** → check **DHCP Server** → **Add Features** → Next.
6. **Features** → Next.
7. **DHCP Server** info page → Next.
8. **Confirmation** → check auto-restart → **Install**. Click **Yes** if prompted to restart.
9. Log back in, click the notification flag → **Complete DHCP configuration**.
10. **Description** → Next.
11. **Authorization** → **Commit**.
12. **Summary** → **Close**.

### Add a DHCP scope

1. Server Manager → **Tools** → **DHCP**.
2. Expand `dc1.yourname.local` → **IPv4**.
3. Right-click **IPv4** → **New Scope** (if IPv4 shows red instead of green, right-click it → **Authorize** first, then refresh).
4. **New Scope Wizard** → Next.
5. Name it **Scope1** → Next.
6. Set the IP range: Start `192.168.0.1`, End `192.168.0.254`, Length `24`, Subnet Mask `255.255.255.0` → Next.
7. Under Exclusions, exclude `192.168.0.1`–`192.168.0.10` → **Add** → Next.
8. **Lease Duration** → leave default → Next.
9. **Configure DHCP Options** → leave default → Next.
10. **Router (Default Gateway)** → Next.
11. **Domain Name and DNS Servers** → confirm `192.168.0.10` is listed → Next.
12. **WINS Servers** → Next.
13. **Activate Scope** → Next → **Finish**.

### Configure RAID 5 for extra storage

1. Shut down the VM.
2. Select it → **Hardware** → **Add** → **Hard Disk**. Set size to **5 GB** → **Add**.
3. Repeat two more times so you have **three 5 GB virtual disks** total attached to the VM.
4. Start the VM and log in.
5. Open **Computer Management** → **Disk Management**. Choose **GPT** when prompted to initialize the new disks.
6. Right-click the first disk → **New RAID-5 Volume** → Next.
7. Move all available disks to the "Selected" column → Next.
8. Assign a drive letter (e.g. `E:`) → Next.
9. Label the volume **DATA**, check "Perform a quick format" → Next → **Finish**. Confirm the warning prompt with **Yes**.

---

## Part 3: Create the Windows 11 Pro Virtual Machine

### Create and launch the VM

1. In the Proxmox web UI, click **Create VM**.
2. **General** tab: give it a VM ID and set Name to **Win11Pro**.
3. **OS** tab: select **Use CD/DVD disc image file (ISO)**, choose the Windows 11 ISO from storage, and set Guest OS Type to **Microsoft Windows** / version **11/2022/2025**.
4. **System** tab: set BIOS to **OVMF (UEFI)** and enable a **TPM state** disk (v2.0) — required for Windows 11 — and enable **Qemu Agent**.
5. **Disks** tab: set the disk size to **64 GB**.
6. **CPU** tab: leave defaults (2 cores).
7. **Memory** tab: set to **4096 MB**.
8. **Network** tab: leave the bridge as `vmbr0` for now (you'll isolate it in the next step).
9. **Confirm** tab: review and click **Finish**.
10. Select the VM → **Start**, then open the **Console**.
11. Windows installs (the boot screen may briefly say "Server" — that's expected and can be ignored). Choose **Windows 11 Pro** as the edition when prompted.
12. Let the VM restart as many times as needed until you reach your user profile, using credentials `Student` / `Password1` if prompted.
13. To power off, shut down from inside Windows rather than using Proxmox's **Stop** button.

### Isolate the network adapter

1. Select the VM → **Hardware** → double-click **Network Device**.
2. Set the bridge to the same internal-only bridge you created for the Windows Server VM (e.g. `vmbr1`), so both VMs can reach each other but stay off your production network. Click **OK**.

### Join the VM to the domain

1. Turn on the Windows 11 VM.
2. Open **CMD** and run `ipconfig` to confirm it received an IP from the DHCP scope (should start after `192.168.0.10`).
3. Search **About your PC** and open it.
4. Under Related Links, click **Domain or workgroup**.
5. Click **Change**.
6. Under "Member of," select the **Domain** radio button and enter your domain name (e.g. `yourname.local`).
7. When prompted, enter your Windows Server administrator credentials. You should see a "Welcome to the yourname.local" confirmation → **OK**.
8. Click **OK** on the restart notice.
9. Click **Close**, then restart the VM.
10. After restart, click **Other user** and log in with one of the AD users you created (e.g. `yourname.local\user1`).
11. You're now logged into the domain-joined Windows 11 Pro VM as an Active Directory user.

### Configure RAID 0 (mirrored) for extra storage

1. Make sure the Windows 11 Pro VM is powered off.
2. Select it → **Hardware** → **Add** → **Hard Disk**. Set size to **5 GB** → **Add**. Repeat once more so you have **two 5 GB drives** total.
3. Start the VM and log in.
4. Search **Computer Management** and open it.
5. In Disk Management, confirm **GPT** is selected when the Initialize Disk prompt appears.
6. Right-click the first unallocated drive → **New Mirrored Volume**.
7. **New Mirrored Volume Wizard** → Next.
8. Highlight the available drive → **Add** to move both disks into the Selected column → Next.
9. Assign a drive letter (e.g. `F:`) → Next.
10. Label the volume **DATA**, check "Perform a quick format" → Next → **Finish**. Confirm with **Yes**.
11. RAID 0 mirroring is now configured on the Windows 11 Pro VM.

---

## Notes

- Proxmox's **Thin Provision** disk option lets a virtual disk start small and grow dynamically up to its maximum size, similar to not pre-allocating full size — leave it enabled unless you specifically need pre-allocated storage.
- Check your host's virtualization requirements before deploying resource-heavy ISOs — verify VT-x/AMD-V is enabled in BIOS/UEFI.
- Reference on the DHCP handshake process: [Understanding the DORA Process in DHCP](https://ipwithease.com/understanding-dora-process-in-dhcp/)
- **Group Policy (GPO):** on DC1, right-click your domain → **Create a GPO** as needed for policy management (e.g. mirroring what your IPAM server enforces).
