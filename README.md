# Windows Server 2019 Full Lab — Active Directory, GPO, DHCP, DNS & File Services

A hands-on home lab covering the core responsibilities of a Windows Server / Active Directory administrator: forest and domain setup, OU and group design, Group Policy hardening, DHCP failover, DNS load balancing, file server permissions with quotas and file screening, print management, Hyper-V, and automated backups.

> 📸 **Screenshots**: Placeholders are marked throughout as `![description](images/xx-name.png)`. Drop your screenshots into an `images/` folder in this repo using matching filenames, or rename the placeholders to match your own filenames.

---

## Lab Topology

| Role | Hostname | IP Address | OS |
|---|---|---|---|
| Primary Domain Controller | **PDC** | 192.168.1.2 /24 | Windows Server 2019 (Desktop Experience) |
| Additional Domain Controller | **ADC** | 192.168.1.3 /24 | Windows Server 2019 (Server Core) |
| Gateway | — | 192.168.1.1 | — |
| Domain | **FINAL.LOCAL** | — | — |

![Lab topology diagram](images/00-topology.png)

---

## Naming Conventions Used in This Guide

Adjust these to match your own environment if different — they're referenced consistently throughout:

- **OUs:** `HR`, `Sales`, `Dev`, `IT`
- **Security groups:** `G_HR`, `G_Sales`, `G_Dev`, `G_IT`
- **Shared folders:** `D:\Shares\Dev`, `D:\Shares\HR`, `D:\Shares\Public`
- **Mapped drive letters:** Dev → `Z:`, HR → `Y:`, Public → `X:`
- **Printer name:** `BW-Printer`

---

## Table of Contents

1. [Create the Domain Environment (FINAL.LOCAL)](#1-create-the-domain-environment-finallocal)
2. [Create OUs for Each Department](#2-create-ous-for-each-department)
3. [Create Users and Groups per Department](#3-create-users-and-groups-per-department)
4. [Configure Password Policy](#4-configure-password-policy)
5. [Configure Account Lockout Policy](#5-configure-account-lockout-policy)
6. [Enable Remote Desktop via Group Policy](#6-enable-remote-desktop-via-group-policy)
7. [Allow Remote Assistance with IT as Helpers](#7-allow-remote-assistance-with-it-as-helpers)
8. [Prohibit External Storage, Task Manager, and Control Panel](#8-prohibit-external-storage-task-manager-and-control-panel)
9. [Create the DHCP Scope](#9-create-the-dhcp-scope)
10. [Configure DHCP Failover and Promote ADC](#10-configure-dhcp-failover-and-promote-adc)
11. [Install Web Server (IIS) on PDC and Core](#11-install-web-server-iis-on-pdc-and-core)
12. [DNS Load Balancing for www.final.local](#12-dns-load-balancing-for-wwwfinallocal)
13. [Shared Folder for Dev & HR — Create/Edit, No Delete, File Screening](#13-shared-folder-for-dev--hr--createedit-no-delete-file-screening)
14. [Mapped Network Drive for Dev and HR](#14-mapped-network-drive-for-dev-and-hr)
15. [Mapped Network Drive for Public Folder with Quota](#15-mapped-network-drive-for-public-folder-with-quota)
16. [Add Printer with B&W-Only and Time Restrictions](#16-add-printer-with-bw-only-and-time-restrictions)
17. [Enable Active Directory Recycle Bin](#17-enable-active-directory-recycle-bin)
18. [Install Hyper-V and Configure Replica](#18-install-hyper-v-and-configure-replica)
19. [Join a Client Machine to the Domain](#19-join-a-client-machine-to-the-domain)
20. [Daily Full Server Backup to ADC](#20-daily-full-server-backup-to-adc)

---

## 1. Create the Domain Environment (FINAL.LOCAL)

**Goal:** Stand up a new AD forest on the server that will become the PDC, with static IP `192.168.1.2/24` and gateway `192.168.1.1`.

**Steps:**
1. Set a static IP on the server: `192.168.1.2`, subnet `255.255.255.0`, gateway `192.168.1.1`, and point DNS at itself (`127.0.0.1`, updated automatically once AD DS is installed).
2. Install the **Active Directory Domain Services** role via Server Manager → *Add Roles and Features*, or PowerShell:
   ```powershell
   Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
   ```
3. Promote the server: click the flag notification → *Promote this server to a domain controller* → **Add a new forest**.
4. Root domain name: `FINAL.LOCAL`.
5. Set the Forest/Domain Functional Level (highest available), and a DSRM password.
6. Accept the DNS delegation warning (expected — no parent zone exists).
7. Review and install. The server reboots automatically.

   Equivalent PowerShell:
   ```powershell
   Install-ADDSForest `
     -DomainName "FINAL.LOCAL" `
     -DomainNetbiosName "FINAL" `
     -InstallDNS `
     -SafeModeAdministratorPassword (ConvertTo-SecureString "YourDSRMPassword!" -AsPlainText -Force) `
     -Force
   ```

![Promoting the server to a Domain Controller](images/01-promote-dc.png)

---

## 2. Create OUs for Each Department

**Goal:** Organize objects logically and enable department-scoped Group Policy and delegation.

**Steps:**
1. Open **Active Directory Users and Computers** (`dsa.msc`).
2. Right-click the domain root `FINAL.LOCAL` → *New* → *Organizational Unit*.
3. Create four OUs: `HR`, `Sales`, `Dev`, `IT`.
4. Leave **"Protect container from accidental deletion"** checked (default, recommended).

   PowerShell equivalent:
   ```powershell
   New-ADOrganizationalUnit -Name "HR"    -Path "DC=final,DC=local"
   New-ADOrganizationalUnit -Name "Sales" -Path "DC=final,DC=local"
   New-ADOrganizationalUnit -Name "Dev"   -Path "DC=final,DC=local"
   New-ADOrganizationalUnit -Name "IT"    -Path "DC=final,DC=local"
   ```

![Organizational Units created in ADUC](images/02-ous.png)

---

## 3. Create Users and Groups per Department

**Goal:** Each department gets its own users and a security group that those users belong to, for permission and GPO targeting.

**Steps:**
1. Inside each department OU, right-click → *New* → *User* and create the relevant accounts (set a strong temporary password, and check "User must change password at next logon").
2. Right-click the OU → *New* → *Group* → create a **Global Security Group** named `G_HR`, `G_Sales`, `G_Dev`, `G_IT` respectively.
3. Add each department's users as members of its matching group (double-click the group → *Members* tab → *Add*).

   PowerShell equivalent (example for HR):
   ```powershell
   New-ADGroup -Name "G_HR" -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=final,DC=local"
   New-ADUser -Name "Sara Ahmed" -SamAccountName "sara.ahmed" -Path "OU=HR,DC=final,DC=local" -Enabled $true -AccountPassword (ConvertTo-SecureString "P@ssw0rd1" -AsPlainText -Force)
   Add-ADGroupMember -Identity "G_HR" -Members "sara.ahmed"
   ```

![Users and groups inside the HR OU](images/03-users-groups.png)

---

## 4. Configure Password Policy

**Goal:** Passwords expire every 60 days, minimum length 6 characters, complexity required, and the last 3 passwords remembered.

**Steps:**
1. Open **Group Policy Management** → expand *Domain Security Policy* (or edit the **Default Domain Policy**, since password policy applies at the domain level only).
2. Navigate to *Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy*.
3. Set:
   - **Maximum password age:** `60 days`
   - **Minimum password length:** `6 characters`
   - **Password must meet complexity requirements:** `Enabled`
   - **Enforce password history:** `3 passwords remembered`
4. Run `gpupdate /force` on the DC to apply immediately.

> **Note:** Fine-grained password policies (PSOs) exist if you need *different* policies per OU/group — not required here since the domain-wide policy covers it.

![Default Domain Policy password settings](images/04-password-policy.png)

---

## 5. Configure Account Lockout Policy

**Goal:** Lock an account for 1 hour after 4 failed password attempts.

**Steps:**
1. In the same **Default Domain Policy**, go to *Account Policies → Account Lockout Policy*.
2. Set:
   - **Account lockout threshold:** `4 invalid logon attempts`
   - **Account lockout duration:** `60 minutes`
   - **Reset account lockout counter after:** `60 minutes` (or less — must be ≤ lockout duration)
3. `gpupdate /force` to apply.

![Account lockout policy settings](images/05-lockout-policy.png)

---

## 6. Enable Remote Desktop via Group Policy

**Goal:** Turn on RDP for all domain computers centrally, rather than machine-by-machine.

**Steps:**
1. In Group Policy Management, create a new GPO (e.g. `Enable-RDP-AllComputers`) and link it at the domain root.
2. Edit → *Computer Configuration → Policies → Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host → Connections*.
3. Enable **"Allow users to connect remotely by using Remote Desktop Services."**
4. Under *Windows Defender Firewall with Advanced Security → Inbound Rules*, enable the predefined **Remote Desktop – User Mode (TCP-In)** and **(UDP-In)** rules (port 3389).
5. Under *Computer Configuration → Preferences → Control Panel Settings → Local Users and Groups*, add the relevant group (e.g. `G_IT`, or `Domain Users` if everyone should have access) to the local **Remote Desktop Users** group on target machines.
6. `gpupdate /force`, then verify with `gpresult /r` and a test RDP connection.

![RDP GPO setting enabled](images/06-rdp-gpo.png)

---

## 7. Allow Remote Assistance with IT as Helpers

**Goal:** Enable Remote Assistance domain-wide, with the IT group authorized as the designated helpers.

**Steps:**
1. In the same or a new GPO, go to *Computer Configuration → Policies → Administrative Templates → System → Remote Assistance*.
2. Enable **"Configure Offer Remote Assistance"** (for unsolicited help) and/or **"Configure Solicited Remote Assistance"** depending on whether IT should be able to proactively connect or only respond to user requests.
3. Click *Show* next to *Helpers* and add `FINAL\G_IT`.
4. Ensure the firewall allows Remote Assistance (predefined firewall group **"Remote Assistance"**).
5. Link the GPO domain-wide, then `gpupdate /force` and test from an IT account.

![Remote Assistance helpers configuration](images/07-remote-assistance.png)

---

## 8. Prohibit External Storage, Task Manager, and Control Panel

**Goal:** Lock down regular users from removable storage, Task Manager, and Control Panel — while keeping IT admins unaffected.

**Steps:**
1. Create a GPO (e.g. `Restrict-Storage-TaskMgr-ControlPanel`) linked to the department OUs (HR/Sales/Dev), **not** IT.
2. *Computer Configuration → Policies → Administrative Templates → System → Removable Storage Access* → Enable **"All Removable Storage classes: Deny all access."**
3. *User Configuration → Policies → Administrative Templates → System → Ctrl+Alt+Del Options* → Enable **"Remove Task Manager."**
4. *User Configuration → Policies → Administrative Templates → Control Panel* → Enable **"Prohibit access to Control Panel and PC settings."**
5. If linking at the domain root instead of per-OU, use **Security Filtering** (GPO → Scope tab) to remove `Authenticated Users` and add only the restricted groups, keeping IT/Admins excluded.
6. `gpupdate /force` and test as both a standard user and an IT admin.

> ⚠️ **Lesson learned:** blocking Control Panel access can make some Control Panel applets (e.g. Personalization/theme settings, launched via `rundll32.exe`) fail with a **misleading "Windows cannot access the specified device, path, or file — you may not have the appropriate permissions" error**, instead of a clear "blocked by policy" message. See `TROUBLESHOOTING.md` for how to diagnose this.

![Restricted GPO settings applied](images/08-restrictions-gpo.png)

---

## 9. Create the DHCP Scope

**Goal:** DHCP scope `192.168.1.40`–`192.168.1.230/24`, excluding `.80`–`.85`, with a 10-day lease.

**Steps:**
1. Install the **DHCP Server** role on PDC (or a dedicated server) via Server Manager.
2. Open the DHCP console → right-click IPv4 → *New Scope*.
3. Set the range: Start `192.168.1.40`, End `192.168.1.230`, subnet mask `255.255.255.0`.
4. Add an exclusion range: `192.168.1.80` – `192.168.1.85`.
5. Set the lease duration to `10 days`.
6. Configure scope options (003 Router = `192.168.1.1`, 006 DNS Servers = `192.168.1.2`, `192.168.1.3`).
7. Activate the scope.

   PowerShell equivalent:
   ```powershell
   Add-DhcpServerv4Scope -Name "Final-Lab-Scope" -StartRange 192.168.1.40 -EndRange 192.168.1.230 -SubnetMask 255.255.255.0 -LeaseDuration 10.00:00:00
   Add-DhcpServerv4ExclusionRange -ScopeId 192.168.1.0 -StartRange 192.168.1.80 -EndRange 192.168.1.85
   ```

![DHCP scope configuration](images/09-dhcp-scope.png)

---

## 10. Configure DHCP Failover and Promote ADC

**Goal:** A second server, **ADC** (Server Core, `192.168.1.3`), becomes an Additional Domain Controller and DHCP failover partner.

**Steps:**
1. On ADC (Server Core), install AD DS via PowerShell (no GUI):
   ```powershell
   Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
   Install-ADDSDomainController -DomainName "FINAL.LOCAL" -InstallDNS -Credential (Get-Credential) -SafeModeAdministratorPassword (ConvertTo-SecureString "YourDSRMPassword!" -AsPlainText -Force)
   ```
2. Set ADC's static IP to `192.168.1.3/24`, gateway `192.168.1.1`, preferred DNS pointing to PDC initially.
3. Install the DHCP role on ADC as well.
4. On PDC's DHCP console, right-click the scope → *Configure Failover* → choose ADC as the partner server.
5. Choose **Load Balance** or **Hot Standby** mode depending on your intended failover behavior, and set a shared secret.
6. Verify replication of the scope to ADC in the DHCP console.

![DHCP failover configuration wizard](images/10-dhcp-failover.png)

---

## 11. Install Web Server (IIS) on PDC and Core

**Goal:** Both PDC and ADC (Core) serve the default IIS landing page.

**Steps (PDC — GUI):**
1. Server Manager → *Add Roles and Features* → select **Web Server (IIS)** → Install.
2. Browse to `http://localhost` to confirm the default IIS page loads.

**Steps (ADC — Server Core, PowerShell only):**
```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools
```
Verify from another machine via `http://192.168.1.3`.

![Default IIS page on both servers](images/11-iis-default-page.png)

---

## 12. DNS Load Balancing for www.final.local

**Goal:** `www.final.local` resolves round-robin between `192.168.1.2` and `192.168.1.3`.

**Steps:**
1. Open **DNS Manager** on PDC → *Forward Lookup Zones → final.local*.
2. Right-click → *New Host (A or AAAA)* → Name: `www`, IP: `192.168.1.2` → *Add Host*.
3. Add another host record with the same name `www`, IP: `192.168.1.3` → *Add Host*.
4. Confirm round robin is enabled: right-click the **DNS server name** (not the zone) → *Properties → Advanced tab* → **"Enable round robin"** should be checked (default).
5. Test: `nslookup www.final.local` repeatedly (flush DNS cache between attempts with `ipconfig /flushdns`) — the returned IP should alternate.

   PowerShell equivalent:
   ```powershell
   Add-DnsServerResourceRecordA -ZoneName "final.local" -Name "www" -IPv4Address "192.168.1.2"
   Add-DnsServerResourceRecordA -ZoneName "final.local" -Name "www" -IPv4Address "192.168.1.3"
   ```

> **Note:** This is DNS round robin, not true health-checked load balancing — if one server goes down, DNS will still hand out its IP part of the time.

![DNS round robin records for www.final.local](images/12-dns-roundrobin.png)

---

## 13. Shared Folder for Dev & HR — Create/Edit, No Delete, File Screening

**Goal:** Dev and HR can create and edit files in their shared folders but never delete, and audio/video/executable files are blocked outright.

**Steps:**
1. Create and share `D:\Shares\Dev` and `D:\Shares\HR`. Share permissions: `Change` for the respective group (`G_Dev` / `G_HR`).
2. NTFS permissions (Security tab → Advanced): grant `G_Dev` / `G_HR` — *Create files/write data, Create folders/append data, Read, Read & execute, List folder contents, Write attributes, Write extended attributes*. **Do not** grant Delete.
3. Add an **explicit Deny** on *Delete* and *Delete Subfolders and Files* for the same group. This is required — without it, users can still delete files they personally created due to NTFS default ownership behavior.
4. Install **File Server Resource Manager (FSRM)** (Server Manager → Add Roles and Features).
5. In FSRM → *File Screening Management → File Screens* → *Create File Screen* on each folder path, **Active screening**, blocking the built-in **Audio and Video Files** and **Executable Files** groups.
6. Test as a Dev/HR user: create a file (✅), edit it (✅), delete it (❌ blocked), copy in an `.mp3`/`.exe` (❌ blocked immediately).

![NTFS Deny-Delete permission and FSRM file screen](images/13-shared-folder-permissions.png)

---

## 14. Mapped Network Drive for Dev and HR

**Goal:** Dev and HR each get their shared folder automatically mapped to a drive letter at logon.

**Steps:**
1. Use **Group Policy Preferences**: create/edit a GPO linked to the `Dev` and `HR` OUs.
2. *User Configuration → Preferences → Windows Settings → Drive Maps* → *New → Mapped Drive*.
3. Location: `\\PDC\Dev` (or the FQDN share path), Drive Letter: `Z:` for Dev.
4. Repeat for HR: `\\PDC\HR` → Drive Letter `Y:`.
5. Under the *Common* tab, use **Item-level targeting** to scope each mapping to its respective security group so Dev users only get the Dev drive and vice versa.
6. `gpupdate /force` and log off/on to test.

![Group Policy Drive Maps preference](images/14-mapped-drives-dev-hr.png)

---

## 15. Mapped Network Drive for Public Folder with Quota

**Goal:** All domain users get a mapped public drive, editable by everyone, capped at 2 GB per user.

**Steps:**
1. Create and share `D:\Shares\Public`, granting `Domain Users` Change/Modify at both share and NTFS level.
2. Add the drive map via GPO (as in Task 14) targeting `Domain Users`, mapped to `X:` for everyone — link this GPO at the domain root.
3. In **FSRM → Quota Management → Quotas**, create a quota on `D:\Shares\Public` using the **200 MB/2 GB soft or hard quota template** (or create a custom hard quota of 2 GB) with **"Auto apply template to subfolders"** if each user gets their own subfolder, or a single 2 GB hard quota on the folder if it's shared capacity.
4. Test by copying files as a standard user until the quota blocks further writes.

![FSRM quota configuration on the Public share](images/15-public-drive-quota.png)

---

## 16. Add Printer with B&W-Only and Time Restrictions

**Goal:** All domain users can print, black-and-white only, and only between 9:00 AM and 4:00 PM.

**Steps:**
1. Install the **Print and Document Services** role and add the printer via *Print Management*.
2. Printer *Properties → Advanced* tab → set **Available from 9:00 AM to 4:00 PM**.
3. Printer *Properties → Device Settings* (driver-dependent) → force **Grayscale/Black & White** printing only, or configure this via a **Print Server Properties → Forms/Driver default** if the driver supports enforcing monochrome centrally.
4. Share the printer and set permissions: `Domain Users` → *Print* only (not *Manage this printer*).
5. Optionally deploy via GPO: *Computer Configuration → Preferences → Control Panel Settings → Printers* → *New → Shared Printer* pointing to `\\PDC\BW-Printer`.

![Printer availability time restriction](images/16-printer-restrictions.png)

---

## 17. Enable Active Directory Recycle Bin

**Goal:** Allow recovery of accidentally deleted AD objects.

**Steps:**
1. Open **Active Directory Administrative Center** (`dsac.exe`).
2. Select the domain `final (local)` in the left pane → click **"Enable Recycle Bin"** in the Tasks pane on the right.
3. Confirm the warning (this is irreversible and requires Forest Functional Level 2008 R2 or higher).

   PowerShell equivalent:
   ```powershell
   Enable-ADOptionalFeature -Identity "Recycle Bin Feature" -Scope ForestOrConfigurationSet -Target "final.local"
   ```

![AD Recycle Bin enabled in ADAC](images/17-ad-recycle-bin.png)

---

## 18. Install Hyper-V and Configure Replica

**Goal:** PDC and ADC both run Hyper-V; a Core VM lives on PDC and replicates to ADC.

**Steps:**
1. Install the **Hyper-V** role on both PDC and ADC (Server Manager, or on Core via `Install-WindowsFeature Hyper-V -IncludeManagementTools -Restart`).
2. On PDC, create a new **Server Core VM** via Hyper-V Manager (or `New-VM`), install Windows Server Core inside it.
3. On ADC (the replica target), open Hyper-V Manager → *Hyper-V Settings → Replication Configuration* → **Enable this computer as a Replica server**, and allow authentication (Kerberos for a lab, or certificate-based for production).
4. Open the corresponding firewall rule (**Inbound Rules → Hyper-V Replica HTTP Listener (TCP-In)**).
5. On PDC, right-click the Core VM → *Enable Replication* → point it at ADC → complete the wizard (initial replication method: over the network, external media, or seeded).
6. Verify replication health in Hyper-V Manager's *Replication* tab on both hosts.

![Hyper-V replication configuration between PDC and ADC](images/18-hyperv-replica.png)

---

## 19. Join a Client Machine to the Domain

**Goal:** A Windows client PC joins `FINAL.LOCAL`.

**Steps:**
1. On the client, set DNS to point at `192.168.1.2` (or `.3`) so it can resolve the domain.
2. *Settings → Accounts → Access work or school* (or *System Properties → Computer Name → Change*) → Domain: `FINAL.LOCAL`.
3. Enter domain admin credentials when prompted.
4. Restart when prompted, then log in with a domain account to confirm.

![Client PC joined to FINAL.LOCAL](images/19-client-domain-join.png)

---

## 20. Daily Full Server Backup to ADC

**Goal:** A full backup of the main server (PDC) runs every day at 11:00 PM, with the backup stored on ADC.

**Steps:**
1. Install the **Windows Server Backup** feature on PDC.
2. Ensure the backup destination — a shared folder on ADC (e.g. `\\ADC\Backups`) — exists and PDC's computer account has write permission to it.
3. Open **Windows Server Backup** → *Backup Schedule Wizard*.
4. Choose **Full Server (recommended)** backup configuration.
5. Set the schedule time to `11:00 PM` daily.
6. Choose **"Back up to a shared network folder"** and specify `\\ADC\Backups`.
7. Complete the wizard and verify the first scheduled run completes successfully (check *Windows Server Backup → Status*, or Event Viewer).

   PowerShell equivalent (registering a scheduled backup policy):
   ```powershell
   $policy = New-WBPolicy
   Add-WBBareMetalRecovery $policy
   Add-WBSystemState $policy
   $target = New-WBBackupTarget -NetworkPath "\\ADC\Backups"
   Add-WBBackupTarget -Policy $policy -Target $target
   Set-WBSchedule -Policy $policy -Schedule 23:00
   Set-WBPolicy -Policy $policy
   ```

![Windows Server Backup schedule configuration](images/20-server-backup.png)

---

## Repo Structure

```
├── README.md              # This file — full step-by-step lab guide
├── TROUBLESHOOTING.md      # Real issues hit during the build and how they were diagnosed/fixed
└── images/                 # Screenshots referenced above (add your own)
```

See [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md) for real debugging scenarios encountered while building this lab.
