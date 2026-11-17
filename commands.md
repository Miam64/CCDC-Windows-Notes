# CCDC Windows Command Reference

Plain-text commands for the Windows defense checklist. Every code block uses straight quotes and no special characters, so it is safe to copy and paste into PowerShell.

## How to use this

- Run PowerShell **as Administrator**.
- Replace names, paths, and IPs with your own (`bob`, `corp.local`, `10.0.0.5`, `Docs`).
- Read every command before running it. Know what it does and be ready to explain it.
- Test in your lab first. Take a backup or snapshot before running anything marked **CHANGES SYSTEM**.
- Confirm with your coach that scripts and internet access are allowed at your tryout.

**Labels**

- **READ-ONLY**: lists or checks things, changes nothing. Safe to run.
- **CHANGES SYSTEM**: modifies, disables, deletes, or restores something. Can break scored services. Think first, change one thing at a time, and re-test.

---

## 1. Setup and backups (do first)

**CHANGES SYSTEM** (creates a folder). Keep it outside any share and outside inetpub.

```powershell
New-Item -ItemType Directory C:\backup -Force
```

### Shares

**READ-ONLY**. Note Name and Path. Ignore hidden defaults (C$, ADMIN$, IPC$).

```powershell
Get-SmbShare | Select Name,Path,Special
Get-SmbShare | Get-SmbShareAccess
```

**CHANGES SYSTEM** (writes backup files only). Zip every non-default share and save names and permissions.

```powershell
Get-SmbShare | Where-Object { -not $_.Special } | ForEach-Object { Compress-Archive -Path $_.Path -DestinationPath "C:\backup\$($_.Name).zip" -Force }
Get-SmbShare | Export-Csv C:\backup\shares.csv -NoTypeInformation
Get-SmbShare | Get-SmbShareAccess | Export-Csv C:\backup\share-access.csv -NoTypeInformation
```

Zip one share by hand (use the Path from Get-SmbShare):

```powershell
Compress-Archive -Path C:\Shares\Docs -DestinationPath C:\backup\Docs.zip
```

If zip fails (large folder or files in use), copy it and keep NTFS permissions:

```powershell
robocopy C:\Shares\Docs C:\backup\Docs /E /COPYALL
```

**CHANGES SYSTEM**. Restore a share or recreate a deleted one:

```powershell
Expand-Archive C:\backup\Docs.zip -DestinationPath C:\Shares
New-SmbShare -Name Docs -Path C:\Shares\Docs
```

### IIS (inetpub has no space in the path)

**CHANGES SYSTEM** (writes backup files only):

```powershell
Import-Module WebAdministration
Backup-WebConfiguration -Name pre-change
Copy-Item C:\inetpub C:\backup\inetpub -Recurse
```

Alternative that keeps permissions:

```powershell
robocopy C:\inetpub C:\backup\inetpub /E /COPYALL
```

### MySQL

**READ-ONLY**. Find mysqldump (it is often not in PATH):

```powershell
Get-Command mysqldump -ErrorAction SilentlyContinue
```

If nothing comes back, look in `C:\Program Files\MySQL\MySQL Server 8.0\bin` (version may differ) or `C:\xampp\mysql\bin`.

**CHANGES SYSTEM** (writes a backup file only). It prompts for the password, so the password stays out of your history. Use --result-file, not `>`, because PowerShell can write redirected output in the wrong encoding.

```powershell
$bin = "C:\Program Files\MySQL\MySQL Server 8.0\bin"
& "$bin\mysqldump.exe" -u root -p --all-databases --result-file=C:\backup\all.sql
Get-Item C:\backup\all.sql
```

**CHANGES SYSTEM**. Restore (PowerShell has no `<`, so go through cmd):

```powershell
cmd /c '"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" -u root -p < C:\backup\all.sql'
```

### DNS

**CHANGES SYSTEM** (writes a backup file only):

```powershell
Export-DnsServerZone -Name corp.local -FileName corp.local.bak
```

---

## 2. Accounts and passwords

**READ-ONLY**

```powershell
Get-LocalUser | Select Name,Enabled,PasswordExpires,PasswordRequired
Get-LocalGroupMember Administrators
Get-ADUser -Filter * -Properties PasswordNeverExpires | Where PasswordNeverExpires
Get-ADGroupMember "Domain Admins"
Get-ADGroupMember "Enterprise Admins"
Get-ADGroupMember "Schema Admins"
```

**CHANGES SYSTEM**. Ask your team how to handle scoring service accounts before changing any password.

```powershell
Set-LocalUser -Name bob -Password (Read-Host -AsSecureString)
Disable-LocalUser -Name Guest
Remove-LocalGroupMember -Group Administrators -Member bob
Set-ADAccountPassword -Identity bob -Reset -NewPassword (Read-Host -AsSecureString)
```

---

## 3. Services and processes

**READ-ONLY**

```powershell
Get-Service W3SVC,LanmanServer,MySQL,DNS,NTDS | ft Name,Status,StartType
Get-Service | Where Status -eq Running
sc.exe qc SomeSvc
Get-Process | Sort CPU -Descending | Select -First 15
Get-CimInstance Win32_Process | Select Name,ProcessId,ParentProcessId,CommandLine
Get-AuthenticodeSignature C:\path\file.exe
Get-WinEvent -FilterHashtable @{LogName='System';Level=2;StartTime=(Get-Date).AddHours(-1)}
```

**CHANGES SYSTEM**. Identify which services are scored before stopping or disabling anything.

```powershell
Start-Service W3SVC
Restart-Service W3SVC
Set-Service -Name SomeSvc -StartupType Disabled
```

---

## 4. Network and firewall

**READ-ONLY**

```powershell
Get-NetTCPConnection -State Listen | Select LocalAddress,LocalPort,OwningProcess
Get-NetTCPConnection -State Listen | Select LocalPort,@{n='Proc';e={(Get-Process -Id $_.OwningProcess).Name}}
Get-NetTCPConnection -State Established
Test-NetConnection 10.0.0.5 -Port 80
Get-NetFirewallProfile | Select Name,Enabled,DefaultInboundAction,DefaultOutboundAction
Get-NetFirewallRule -Direction Inbound -Enabled True | Select DisplayName,Action
```

**CHANGES SYSTEM**. List what the scored services need first. A wrong firewall change can cut your own RDP session or take down a service.

```powershell
New-NetFirewallRule -DisplayName "Allow HTTP" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
Disable-NetFirewallRule -DisplayName "Rule Name"
```

**CHANGES SYSTEM, HIGH RISK.** Blocks all inbound traffic that has no allow rule. Add your allow rules (and RDP, if you need it) first.

```powershell
Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultInboundAction Block
```

---

## 4a. simplewall

simplewall is a free third-party firewall. It is **not** a front end for Windows Firewall. It configures the Windows Filtering Platform directly and runs independently alongside Windows Firewall. If either one blocks a connection, it is blocked, so check both when a service breaks. Keep Windows Firewall on as a second layer.

How it behaves:

- It blocks all apps by default (whitelist style), so you allow only what is needed.
- Its filters stay active even after simplewall is closed. Use the **Disable filters** button to remove them.
- Rules can be permanent or temporary. Temporary rules reset at the next reboot, which is a useful safety net while you learn.
- Custom rules are **global** (all apps) or **special** (chosen apps, set on the rule's Apps tab).

Workflow:

1. Finish backups and list each scored service's ports and programs first.
2. Open simplewall but do not enable filters yet. Create your allow rules first.
3. Prefer temporary rules while learning.
4. Enable filters, then test every service from another machine after each rule.
5. Know your rollback before you start: Disable filters, or reboot if rules are temporary.

**CHANGES SYSTEM, HIGH RISK.** Command-line options (run from the folder containing simplewall.exe). Enabling filters with no allow rules can cut off RDP, DNS, and the scoring checks immediately.

```powershell
.\simplewall.exe -install -temp
.\simplewall.exe -install
.\simplewall.exe -uninstall
```

- `-install -temp` enables filtering until reboot.
- `-install` enables filtering.
- `-uninstall` removes all filters simplewall installed.

**READ-ONLY.** Dump the WFP filters and state to inspect what is actually active (writes files to the current folder):

```powershell
netsh wfp show filters
netsh wfp show state
```

### What to allow, by machine role

Use only the rows for your machine. Services run by the System process (IIS, SMB) are likely easier to allow with a port-based rule than a per-program rule. Verify this in your lab.

| Service | Inbound allow | Notes |
|---|---|---|
| IIS | TCP 80 (443 if HTTPS is scored) | |
| SMB / file shares | TCP 445 | 139 is legacy, skip unless needed |
| DNS server | TCP and UDP 53 | |
| MySQL | TCP 3306 | Limit to the web server IP if possible |
| RDP | TCP 3389 | Only if needed, ideally from specific IPs |
| Domain controller | 53, 88, 135, 389, 445, 464, 636, 3268/3269, dynamic RPC | Get the exact list from your team first |

Outbound, allow only what the machine starts itself:

- DNS (53) to your DNS server, and DHCP if used
- NTP (123) for time sync, which matters for AD
- Web server to the database host on 3306
- Domain members to the DC (the AD ports above)
- Windows Update, only if allowed and safe

The firewall is stateful, so replies to connections you allowed inbound need no separate outbound rule.

Ask your team for their exact simplewall setup and copy it. Practice in your lab until you can enable, test, and disable it quickly.

---

## 5. Shares and SMB hardening

**CHANGES SYSTEM**. Disable SMBv1 only if nothing needs it.

```powershell
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

---

## 6. IIS

**READ-ONLY**

```powershell
Import-Module WebAdministration
Get-Website
Get-WebBinding
Get-ChildItem IIS:\AppPools
icacls C:\inetpub\wwwroot
```

**CHANGES SYSTEM**

```powershell
Start-Website "Default Web Site"
Start-WebAppPool DefaultAppPool
iisreset
```

---

## 7. MySQL

**READ-ONLY**

```powershell
mysql -u root -p -e "SELECT user,host FROM mysql.user;"
```

---

## 8. DNS and Active Directory

**READ-ONLY**

```powershell
Get-DnsServerZone
Get-DnsServerResourceRecord -ZoneName corp.local
Resolve-DnsName www.corp.local -Server 127.0.0.1
dcdiag /q
repadmin /replsummary
w32tm /query /status
Get-GPO -All | Select DisplayName,ModificationTime
```

**CHANGES SYSTEM**

```powershell
Add-DnsServerResourceRecordA -Name www -ZoneName corp.local -IPv4Address 10.0.0.5
```

---

## 9. Persistence hunting

All **READ-ONLY**. Investigate before deleting anything, and note what you find.

```powershell
Get-ScheduledTask | Where State -ne Disabled | Select TaskName,TaskPath
Get-CimInstance Win32_StartupCommand | Select Name,Command,Location
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Run
Get-ItemProperty HKCU:\Software\Microsoft\Windows\CurrentVersion\Run
Get-CimInstance -Namespace root/subscription -ClassName __EventConsumer
Get-Content $PROFILE.AllUsersAllHosts
Get-ChildItem Cert:\LocalMachine\Root | Sort NotBefore -Descending | Select -First 10 Subject,Thumbprint,NotBefore
Get-ChildItem C:\Users -Recurse -Include *.exe -Force -ErrorAction SilentlyContinue
```

IFEO (image hijack) check. Look for Debugger values (for example sethc.exe pointing to cmd.exe):

```powershell
$k = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
Get-ItemProperty "$k\Image File Execution Options\*" | Where Debugger
```

---

## 9a. Autoruns (Sysinternals)

Autoruns lists nearly everything that runs at startup or logon. It does not show everything, so use it alongside Task Scheduler, services, and Process Explorer.

**Get it:** download from the Microsoft Sysinternals site, or use a tools folder your team provides. Ask your coach whether downloads and internet are allowed. Use `autoruns64.exe` (GUI) and `autorunsc64.exe` (command line, good for Server Core). Run as administrator or some entries will be missing.

### GUI workflow

1. Save a baseline early: File > Save. Later use File > Compare to see only new entries (shown in green).
2. Start on the **Everything** tab, then check **Logon**, **Services**, **Scheduled Tasks**, **Drivers**, **Image Hijacks**, **WMI**, **Winlogon**, **AppInit**, and **Boot Execute**.
3. Options > Scan Options > turn on **Verify code signatures**. **Check VirusTotal.com** sends file hashes to the internet, so use it only if you have internet and it is allowed.
4. Investigate every highlighted row:
   - Pink: no publisher or an invalid signature.
   - Yellow: the entry exists but its file is missing.
   - Green: new since your saved baseline.
   - Red detection counts in the VirusTotal column are bad.
5. Click an entry to see its path, publisher, and command line in the bottom pane. Right-click for Jump to Entry (registry location), Jump to Image (file location), Properties, and Search Online.
6. **Disable before deleting.** Uncheck an entry to disable it (reversible), then re-test your scored services. Delete it only once you are confident it is malicious.
7. Press F5 to refresh regularly, because attackers re-add persistence.

### What looks suspicious

- No publisher, random file names, or files in Temp, AppData, ProgramData, Public, or Users folders.
- Script files (.ps1, .vbs, .bat) or encoded PowerShell commands.
- `cmd.exe`, `powershell.exe`, `mshta.exe`, `rundll32.exe`, or `wscript.exe` with odd arguments.
- Image Hijacks (IFEO) entries, such as sethc.exe pointing to cmd.exe. Delete these.
- Drivers you do not recognize.

Hide Microsoft Entries reduces noise, but attackers abuse signed Windows programs, so also look at the full list. Never remove entries for scored services (IIS, MySQL, DNS, and so on). If you find a malicious signed file, check its certificate and see whether other files use it.

### Command line (autorunsc)

**READ-ONLY** for the system (it writes one output file). Saves a CSV of non-Microsoft entries with file hashes and signature checks. Run it again later and compare the two files. Going through cmd avoids PowerShell's output encoding problem.

```powershell
cmd /c "autorunsc64.exe -accepteula -a * -c -h -s -m -nobanner > C:\backup\autoruns.csv"
```

Compare two saved runs:

```powershell
Compare-Object (Get-Content C:\backup\autoruns.csv) (Get-Content C:\backup\autoruns-later.csv)
```

---

## 10. Event logs

**READ-ONLY**. 4624 is a successful login and 4625 is a failed login. A flood of 4625 suggests brute force.

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4625} -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4624} -MaxEvents 20
```

---

## 11. Advanced

**CHANGES SYSTEM, HIGH RISK.** Enables PPL for LSASS. Needs a reboot. Test in your lab first.

```powershell
Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa -Name RunAsPPL -Value 1 -Type DWord
```

Server Core configuration menu:

```powershell
sconfig
```

---

## Troubleshooting order for a broken service

1. Is the service running?
2. Is it listening on the right port?
3. Does the firewall allow it?
4. Do the file and account permissions allow it?
5. Are the config and bindings correct?
6. Is DNS resolving?

Make one change at a time, re-test after each, and log what broke, the cause, and the fix.
