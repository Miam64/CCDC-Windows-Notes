# CCDC Tryouts: Windows Defender Playbook

Nov 21, 2026. AIEB 203. Arrive 8:20 AM. Start 9:00 AM. Lunch and Red Team switch 12:00 PM. End 4:00 PM.

Open this on your own device, next to the remote session. Jump to the phase you are in.

- [0. Rules that can disqualify you](#0-rules-that-can-disqualify-you)
- [1. Hosts and services](#1-hosts-and-services)
- [2. Arrival checklist (8:20 AM)](#2-arrival-checklist-820-am)
- [3. PHASE 1 (9:00 to 12:00): keep services up](#3-phase-1-900-to-1200-keep-services-up)
- [4. Per-service quick fixes](#4-per-service-quick-fixes)
- [5. PHASE 2 (12:00 to 4:00): hunt and clean up](#5-phase-2-1200-to-400-hunt-and-clean-up)
- [6. Log and incident report](#6-log-and-incident-report)
- [7. Where things live](#7-where-things-live)

**Command labels**

- **READ-ONLY**: lists or checks things. Changes nothing.
- **CHANGES SYSTEM**: modifies, disables, deletes, or restores something. Can break scored services. One change at a time, then re-test.
- **AFTER NOON ONLY**: counts as threat hunting. Do not run before 12:00 PM.

Run PowerShell **as Administrator**. Replace names, paths, and IPs with your own.

---

## 0. Rules that can disqualify you

1. **No threat hunting and no removing red team implants before 12:00 PM.** Honesty policy. If you started early, you can be disqualified. Before noon, fix and maintain services only.
2. **Prohibited tools: simplewall, Malwarebytes, LLM chatbots, and AI agents of any kind.** The only exception is Google's search AI overview. Use the built-in Windows Defender Firewall (`wf.msc`).
3. **Do not remove malware artifacts or indicators of compromise from the environment.** No saving them to your personal computer, no uploading to VirusTotal, and no tools that upload to antivirus companies. Keep the VirusTotal option **OFF** in Autoruns and Process Explorer.
4. **Keep the `whiteteam` account active and unchanged.** Do not rotate its password and do not remove its admin rights.
5. **Do not remove whitelisted binaries or block whitelisted IPs.** You may be given a list on the day. Check it before every firewall block and every cleanup.
6. **Stay in your own environment.** Unauthorized access to another competitor's environment is prohibited.
7. **Stay professional** with staff, other competitors, and Red and White Team members.
8. **Reboot and revert are a last resort.** A revert destroys all your changes. See the request template in Section 3.

Scoring is based on the role you signed up for. Service uptime is monitored and scored. Incident response reports must be clear, detailed, and on time.

---

## 1. Hosts and services

X is your team number. Example for team 7: `192.168.7.86`, `team07-ctrl-rm`.

| IP | Hostname | Scored services |
|---|---|---|
| 192.168.X.86 | TeamX-ctrl-rm | ADDS, LDAP, DNS, Remote Access |
| 192.168.X.53 | TeamX-sting | Database, Web, Remote Access |
| 192.168.X.109 | TeamX-certified | Certificates, File Sharing, Remote Access |
| 192.168.X.23 | TeamX-lookup | DNS, Remote Access |
| 192.168.X.11 | TeamX-whale | Web, Database, Credentials, Remote Access |
| 192.168.X.40 | TeamX-andie | Orchestration/HTTP, Remote Access |
| 192.168.X.78 | TeamX-chris | Device Management, Remote Access |
| 192.168.X.99 | TeamX-comms | Mail, Remote Access |

The tryout packet does not say which hosts are Windows. Work that out first (see Section 2). Defend only the Windows hosts you signed up for.

**Connection:** range interface at `https://range-guac-dev-test.tntech.edu/`. Credentials are given on the day. Machines are managed through a Linux jumpbox. Test copy and paste into the remote session early. The Guacamole side menu (Ctrl+Alt+Shift) usually has a clipboard box, but confirm it works before you need it.

---

## 2. Arrival checklist (8:20 AM)

Do these before 9:00.

- [ ] Log in to the range interface and open a session on each machine you own.
- [ ] Note which hosts are Windows (the login screen or `systeminfo` shows the OS).
- [ ] Open a notes file on your own device for your change log (template in Section 6).
- [ ] Test that copy and paste works into the remote session.

**Ask staff** (the packet does not answer these):

- [ ] Are password changes and other hardening allowed before noon, or only service fixes?
- [ ] If a red team implant is breaking a service before noon, what should I do?
- [ ] Where and how do I submit incident reports, and in what format?
- [ ] Are my own notes on my device OK to use?
- [ ] Are Sysinternals tools (Autoruns, Process Explorer, TCPView) already on the machines, or can I download them?
- [ ] Is Windows Defender's automatic sample submission OK, given the rule against uploading artifacts to antivirus companies?

---

## 3. PHASE 1 (9:00 to 12:00): keep services up

**Goal:** every scored service stays up. Fix what breaks. **No threat hunting and no implant removal.**

### 3.1 Routine

1. **Test every scored service** on each Windows host, from another machine if you can. Write down what works at the start.
2. **Back up what you will touch** (Section 3.3).
3. **Fix breaks** with the troubleshooting order below. One change at a time, then re-test.
4. **Log each change** (what, why, when).
5. If something looks malicious, **write it down for noon** and keep fixing the service. Do not remove it yet.

### 3.2 Troubleshooting order (say it out loud)

1. **Is the service running?** `services.msc` or `Get-Service`.
2. **Is it listening on the right port?** `Get-NetTCPConnection -State Listen`.
3. **Does the firewall allow it?** Look for Block rules (commands in Section 3.4).
4. **Permissions?** File, share, or account permissions changed.
5. **Config or bindings?** Something in the app's settings changed.
6. **DNS?** Names must resolve.

**READ-ONLY**

```powershell
Get-Service | Where Status -eq Running | Select Name,DisplayName
Get-NetTCPConnection -State Listen | Select LocalAddress,LocalPort,OwningProcess
Get-NetTCPConnection -State Listen | Select LocalPort,@{n='Proc';e={(Get-Process -Id $_.OwningProcess).Name}}
Test-NetConnection 192.168.X.53 -Port 80
Get-WinEvent -FilterHashtable @{LogName='System';Level=2;StartTime=(Get-Date).AddHours(-1)}
```

**CHANGES SYSTEM**

```powershell
Start-Service SERVICENAME
Restart-Service SERVICENAME
```

### 3.3 Backups before you touch anything

**CHANGES SYSTEM** (creates a folder). Keep it outside any share and outside inetpub. Keep backups inside the environment.

```powershell
New-Item -ItemType Directory C:\backup -Force
```

**Shares.** READ-ONLY listing first. Ignore hidden defaults (C$, ADMIN$, IPC$).

```powershell
Get-SmbShare | Select Name,Path,Special
Get-SmbShare | Get-SmbShareAccess
```

Zip every non-default share and save names and permissions:

```powershell
Get-SmbShare | Where-Object { -not $_.Special } | ForEach-Object { Compress-Archive -Path $_.Path -DestinationPath "C:\backup\$($_.Name).zip" -Force }
Get-SmbShare | Export-Csv C:\backup\shares.csv -NoTypeInformation
Get-SmbShare | Get-SmbShareAccess | Export-Csv C:\backup\share-access.csv -NoTypeInformation
```

If zip fails (large folder or files in use), copy and keep NTFS permissions:

```powershell
robocopy C:\Shares\Docs C:\backup\Docs /E /COPYALL
```

**IIS** (inetpub has no space in the path):

```powershell
Import-Module WebAdministration
Backup-WebConfiguration -Name pre-change
Copy-Item C:\inetpub C:\backup\inetpub -Recurse
```

**DNS zone:**

```powershell
Export-DnsServerZone -Name corp.local -FileName corp.local.bak
```

**MySQL.** Find mysqldump first (often not in PATH):

```powershell
Get-Command mysqldump -ErrorAction SilentlyContinue
```

If nothing comes back, look in `C:\Program Files\MySQL\MySQL Server 8.0\bin` (version may differ) or `C:\xampp\mysql\bin`. It prompts for the password. Use --result-file, not `>`, because PowerShell can write redirected output in the wrong encoding.

```powershell
$bin = "C:\Program Files\MySQL\MySQL Server 8.0\bin"
& "$bin\mysqldump.exe" -u root -p --all-databases --result-file=C:\backup\all.sql
Get-Item C:\backup\all.sql
```

Restore (PowerShell has no `<`, so go through cmd):

```powershell
cmd /c '"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" -u root -p < C:\backup\all.sql'
```

Restore a share or recreate a deleted one:

```powershell
Expand-Archive C:\backup\Docs.zip -DestinationPath C:\Shares
New-SmbShare -Name Docs -Path C:\Shares\Docs
```

### 3.4 Windows Defender Firewall (wf.msc)

Do not use simplewall. Check this layer when a service is unreachable. Never block a whitelisted IP.

**READ-ONLY**

```powershell
Get-NetFirewallProfile | Select Name,Enabled,DefaultInboundAction,DefaultOutboundAction
Get-NetFirewallRule -Direction Inbound -Enabled True -Action Block | Select DisplayName
Get-NetFirewallRule -Direction Inbound -Enabled True -Action Allow | Select DisplayName
```

**CHANGES SYSTEM.** Fix a service that the firewall is blocking:

```powershell
Remove-NetFirewallRule -DisplayName "Rule Name"
New-NetFirewallRule -DisplayName "Allow HTTP" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
Enable-NetFirewallRule -DisplayName "Rule Name"
```

Typical ports: web 80/443, SMB 445, DNS 53 (TCP and UDP), MySQL 3306, RDP 3389, mail 25/587/993, AD 88/389/636/445.

### 3.5 Reboot or revert (last resort)

Post in the **black-team-requests** channel on Discord with all four lines. A revert destroys all your changes.

```
Team Number: X
Machine Hostname: teamXX-hostname
Machine IP Address: 192.168.X.nn
Action Requested: REBOOT   (or REVERT)
```

### 3.6 Ask staff before doing these before noon

- Changing passwords (never `whiteteam`).
- Disabling or deleting accounts.
- Firewall cleanup beyond fixing a broken service.
- Any other hardening.

---

## 4. Per-service quick fixes

Use the card for the services on your host. If the host runs Linux, it is not yours.

### ADDS, LDAP, DNS (ctrl-rm)

Ports: 53, 88, 389, 445, 636. AD depends on DNS and correct time.

**READ-ONLY**

```powershell
Get-Service NTDS,DNS,Netlogon,KDC,W32Time | ft Name,Status,StartType
dcdiag /q
repadmin /replsummary
w32tm /query /status
Get-DnsServerZone
Get-DnsServerResourceRecord -ZoneName corp.local
Resolve-DnsName HOSTNAME -Server 192.168.X.86
Test-NetConnection localhost -Port 389
```

**CHANGES SYSTEM**

```powershell
Start-Service NTDS
Start-Service DNS
w32tm /resync
Add-DnsServerResourceRecordA -Name www -ZoneName corp.local -IPv4Address 192.168.X.53
```

Common breaks: DNS stopped, A records deleted, zone removed, clock skew, firewall blocking 53 or 389.

### DNS (lookup)

Port 53, TCP and UDP.

**READ-ONLY**

```powershell
Get-Service DNS
Get-DnsServerZone
Get-DnsServerResourceRecord -ZoneName corp.local
Get-DnsServerForwarder
Resolve-DnsName HOSTNAME -Server 192.168.X.23
```

**CHANGES SYSTEM:** `Start-Service DNS`, or restore the zone from your export or recreate records with `Add-DnsServerResourceRecordA`.

### Web (IIS) (sting, whale)

Port 80 (and 443 if HTTPS is scored). Service: `W3SVC`.

**READ-ONLY**

```powershell
Import-Module WebAdministration
Get-Service W3SVC
Get-Website
Get-WebBinding
Get-ChildItem IIS:\AppPools
icacls C:\inetpub\wwwroot
```

**CHANGES SYSTEM**

```powershell
Start-Service W3SVC
Start-Website "Default Web Site"
Start-WebAppPool DefaultAppPool
iisreset
```

Common breaks: W3SVC stopped, site or app pool stopped, binding changed (port or hostname), read permission removed on the site folder, default document missing, firewall blocking 80. If you changed files, restore from your inetpub backup.

### Database (sting, whale)

MySQL port 3306.

**READ-ONLY**

```powershell
Get-Service *mysql*
Get-NetTCPConnection -LocalPort 3306 -State Listen
mysql -u root -p -e "SELECT user,host FROM mysql.user;"
```

If it is SQL Server instead, check `Get-Service MSSQLSERVER` and port 1433.

**CHANGES SYSTEM:** `Start-Service MySQL80` (use the real name from the listing). Restore from your dump if data was dropped.

### Certificates and file sharing (certified)

**READ-ONLY**

```powershell
Get-Service CertSvc
certutil -ping
Get-Service LanmanServer
Get-SmbShare | Select Name,Path,Special
Get-SmbShare | Get-SmbShareAccess
Test-NetConnection localhost -Port 445
```

**CHANGES SYSTEM**

```powershell
Start-Service CertSvc
Start-Service LanmanServer
New-SmbShare -Name Docs -Path C:\Shares\Docs
Grant-SmbShareAccess -Name Docs -AccountName Everyone -AccessRight Read -Force
```

Common breaks: share removed, share permissions revoked, files deleted (restore from your zip), CertSvc stopped, firewall blocking 445. Never share C:.

### Mail (comms)

The mail software is not stated. Find it, then check it.

**READ-ONLY**

```powershell
Get-Service | Where DisplayName -match 'mail|smtp|exchange|imap|pop'
Get-NetTCPConnection -State Listen | Where LocalPort -in 25,110,143,465,587,993,995
```

### Remote access (all hosts)

**READ-ONLY**

```powershell
Get-Service TermService
Get-Service WinRM
Get-Service sshd -ErrorAction SilentlyContinue
Get-NetTCPConnection -LocalPort 3389 -State Listen
```

Do not disable the remote access service on a scored host. It is a scored service.

---

## 5. PHASE 2 (12:00 to 4:00): hunt and clean up

Only after 12:00 PM. Keep services up while you clean. Re-test after each removal. **Check the whitelist before you delete or block anything.** Leave `whiteteam` alone.

### 5.1 Order

1. Accounts.
2. Network connections.
3. Persistence (the places attackers hide).
4. Event logs.
5. Hardening.
6. Re-check every 20 to 30 minutes, because red team re-plants.
7. Last 30 minutes: re-test all services and submit your report.

### 5.2 Accounts

**AFTER NOON ONLY. READ-ONLY**

```powershell
Get-LocalUser | Select Name,Enabled,PasswordExpires,PasswordRequired
Get-LocalGroupMember Administrators
Get-ADUser -Filter * -Properties Created | Sort Created -Descending | Select -First 10 Name,Created,Enabled
Get-ADGroupMember "Domain Admins"
Get-ADGroupMember "Enterprise Admins"
Get-ADGroupMember "Schema Admins"
```

**CHANGES SYSTEM.** Never change or remove `whiteteam`. Ask your team how to handle service accounts before changing their passwords.

```powershell
Set-LocalUser -Name bob -Password (Read-Host -AsSecureString)
Disable-LocalUser -Name Guest
Remove-LocalUser -Name baduser
Remove-LocalGroupMember -Group Administrators -Member bob
Set-ADAccountPassword -Identity bob -Reset -NewPassword (Read-Host -AsSecureString)
Disable-ADAccount -Identity baduser
```

### 5.3 Network

**AFTER NOON ONLY. READ-ONLY**

```powershell
Get-NetTCPConnection -State Established | Select LocalPort,RemoteAddress,RemotePort,OwningProcess
Get-NetTCPConnection -State Listen | Select LocalPort,@{n='Proc';e={(Get-Process -Id $_.OwningProcess).Name}}
Get-Process -Id PID | Select Name,Path
```

Look for unexpected listeners and outbound connections to unfamiliar addresses. Map each port to a process and ask whether it belongs. Check IPs against the whitelist before blocking.

**CHANGES SYSTEM**

```powershell
Stop-Process -Id PID -Force
New-NetFirewallRule -DisplayName "Block bad IP" -Direction Outbound -RemoteAddress 1.2.3.4 -Action Block
```

### 5.4 Persistence

**AFTER NOON ONLY. READ-ONLY.** Works with no downloads.

```powershell
Get-ScheduledTask | Where State -ne Disabled | Select TaskName,TaskPath
Get-CimInstance Win32_StartupCommand | Select Name,Command,Location
Get-CimInstance Win32_Service | Where PathName -notmatch 'Windows|Program Files' | Select Name,State,PathName
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Run
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\RunOnce
Get-ItemProperty HKCU:\Software\Microsoft\Windows\CurrentVersion\Run
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" | Select Shell,Userinit
Get-ChildItem "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp" -Force
Get-CimInstance -Namespace root/subscription -ClassName __EventConsumer
Get-CimInstance -Namespace root/subscription -ClassName __EventFilter
Get-Content $PROFILE.AllUsersAllHosts
Get-ChildItem Cert:\LocalMachine\Root | Sort NotBefore -Descending | Select -First 10 Subject,Thumbprint,NotBefore
```

IFEO (image hijack) check. Look for Debugger values (for example sethc.exe pointing to cmd.exe):

```powershell
$k = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
Get-ItemProperty "$k\Image File Execution Options\*" | Where Debugger
```

Recent files in odd places, and possible web shells:

```powershell
Get-ChildItem C:\Users,C:\ProgramData,C:\Windows\Temp -Recurse -Include *.exe,*.ps1,*.bat,*.vbs -Force -ErrorAction SilentlyContinue | Where LastWriteTime -gt (Get-Date).AddHours(-6) | Select FullName,LastWriteTime
Get-ChildItem C:\inetpub\wwwroot -Recurse -Include *.asp,*.aspx,*.ashx,*.php -Force | Sort LastWriteTime -Descending | Select -First 20 FullName,LastWriteTime
```

**CHANGES SYSTEM.** Remove only what you are sure is malicious and not whitelisted.

```powershell
Unregister-ScheduledTask -TaskName "TaskName" -Confirm:$false
sc.exe stop BadSvc
sc.exe delete BadSvc
Remove-ItemProperty -Path HKCU:\Software\Microsoft\Windows\CurrentVersion\Run -Name "BadValue"
Remove-ItemProperty -Path "$k\Image File Execution Options\sethc.exe" -Name Debugger
```

### 5.5 Autoruns (Sysinternals), if available

Ask staff whether it is installed or can be downloaded. It lists nearly everything that runs at startup or logon, but it does not show everything, so use it with the commands above. Run as administrator.

1. File > Save to keep a baseline, then File > Compare later to see new entries (green).
2. Options > Scan Options: turn on **Verify code signatures**. **Leave "Check VirusTotal.com" OFF.** Do not accept the VirusTotal terms prompt.
3. Check these tabs: **Everything**, **Logon**, **Services**, **Scheduled Tasks**, **Drivers**, **Image Hijacks**, **WMI**, **Winlogon**.
4. Investigate highlighted rows. Pink means no publisher or an invalid signature. Yellow means the file is missing.
5. Look for odd paths (Temp, AppData, ProgramData, Public, Users), script files, and `cmd.exe`, `powershell.exe`, `mshta.exe`, `rundll32.exe`, or `wscript.exe` with odd arguments.
6. Uncheck an entry to disable it (reversible), re-test services, and delete only when you are sure.
7. Never remove entries for scored services.

Command-line version (no VirusTotal option). It saves a CSV inside the environment. Running it through cmd avoids PowerShell's output encoding problem.

```powershell
cmd /c "autorunsc64.exe -accepteula -a * -c -s -m -nobanner > C:\backup\autoruns.csv"
```

Process Explorer also has a VirusTotal option. Leave it off. Use it to look at parent and child process chains and injected DLLs.

### 5.6 Event logs

**READ-ONLY.** 4625 is a failed login and 4624 is a successful one. A flood of 4625 suggests brute force. 4720 is a new user, 4732 is a user added to a local group, and 7045 is a new service.

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4625} -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4624} -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4720} -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4732} -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName='System';Id=7045} -MaxEvents 20
```

### 5.7 Hardening (after services are stable)

**CHANGES SYSTEM.** Re-test scored services after each one.

```powershell
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
Set-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name UserAuthentication -Value 1
Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa -Name RunAsPPL -Value 1 -Type DWord
```

The last line enables PPL for LSASS and needs a reboot, so use it with care. Only disable RDP if a host's remote access is not scored, and never on a scored host.

---

## 6. Log and incident report

Keep a running log on your own device. Your report depends on it. Submit the report in the format and place staff tell you.

**Change log line**

```
Time | Host | What I changed | Why | Result (service OK?)
```

**Finding (for the report)**

```
Time found:
Host:
What it was (account, task, service, connection, file):
Where (path, registry key, port):
How I found it:
What I did (removed, disabled, left alone) and why:
Service status after:
```

Describe findings in your report. Do not copy malware, files, hashes, or other indicators out of the environment, and do not upload them anywhere.

---

## 7. Where things live

| Tool | How to open |
|---|---|
| Services | `services.msc` |
| Windows Defender Firewall | `wf.msc` |
| Local Users and Groups | `lusrmgr.msc` |
| AD Users and Computers | `dsa.msc` |
| Task Scheduler | `taskschd.msc` |
| Event Viewer | `eventvwr.msc` |
| IIS Manager | `inetmgr` |
| Shares | `fsmgmt.msc` |
| DNS Manager | `dnsmgmt.msc` |
| Group Policy | `gpmc.msc` (domain), `gpedit.msc` (local) |
| Installed programs | `appwiz.cpl` |
| Advanced system settings | `sysdm.cpl` |
| Registry | `regedit` |
| Server Core menu | `sconfig` |
| Server Manager | `servermanager.exe` |

In File Explorer, turn on View > Show > Hidden items and File name extensions.
