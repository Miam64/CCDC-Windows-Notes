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
