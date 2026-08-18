# TryHackMe — Monitor

**Difficulty:** Medium
**Category:** Web Exploitation → RCE → Credential Harvesting → Privilege Escalation

---

## Overview

Monitor is a web exploitation box centered on an internal NOC monitoring dashboard. The path runs from directory brute-forcing, through a SQL injection auth bypass, into command injection RCE on a "ping" utility, and finally to root via a cracked KeePass database recovered from a backup directory.

**Attack chain summary:**
`Recon → Hidden directory discovery → SQLi auth bypass → RCE via ping utility → Credential leak → SSH foothold → KeePass DB found & cracked → Root`

---

## 1. Reconnaissance

I started with an nmap scan, using verbose mode to watch results come in live rather than waiting for the full scan to complete.

<img width="905" height="557" alt="nmap scan" src="https://github.com/user-attachments/assets/342ff9cf-fde0-42c2-9c4e-4904856e143b" />

**Findings:**
- **Port 22** — OpenSSH (no credentials yet, nothing to act on)
- **Port 5050** — Werkzeug/Python web app

With SSH out of reach for now, I moved to the web app.

<img width="1358" height="1012" alt="landing page" src="https://github.com/user-attachments/assets/6122eeab-4f99-425d-b940-c508bd050800" />

The landing page was completely static — no forms, buttons, or links. Nothing to interact with on the surface, so the next step was enumeration underneath it.

---

## 2. Enumeration

### 2.1 Directory brute-forcing

Running ffuf against the webroot turned up a hidden path not linked anywhere on the page: `/internal`.

<img width="829" height="121" alt="ffuf hidden /internal path" src="https://github.com/user-attachments/assets/780d7484-ea76-4671-adbb-693d9e8ce505" />

Navigating there landed on an operator login portal, gating whatever sat behind it.

<img width="485" height="420" alt="operator login portal" src="https://github.com/user-attachments/assets/1517a6d6-2544-440a-9045-0ee8dc4b2a0d" />

### 2.2 Scoped enumeration behind auth

Before touching the login form, I ran a second ffuf scan scoped to `/internal` to map what else existed behind the gate. This turned up three more endpoints:

- `/internal/health`
- `/internal/logout`
- `/internal/dashboard`

<img width="1054" height="479" alt="scoped ffuf results" src="https://github.com/user-attachments/assets/1444fc48-c7de-44a9-8be9-d9096d0b4aed" />

---

## 3. Initial Access — SQL Injection Auth Bypass

With the app mapped, I manually tested the login form for injection points. The simplest possible SQLi payload was enough to get a response worth investigating.

<img width="1236" height="405" alt="SQLi payload against login form" src="https://github.com/user-attachments/assets/fff36e2a-2415-4596-9f41-0dfb404ffabb" />

The request returned a redirect and a fresh session cookie — the auth check had been bypassed outright. Decoding the JWT in that cookie confirmed the access level it granted.

<img width="1308" height="421" alt="decoded JWT" src="https://github.com/user-attachments/assets/c9b36131-a9c2-4ff9-bfb3-88f9234e1421" />

The token confirmed a valid session as a **NOC operator**.

<img width="1561" height="907" alt="NOC operator dashboard access" src="https://github.com/user-attachments/assets/f7756d34-1d44-4668-96c5-00f1d063638c" />

**Dashboard recon:**
- The audit log leaked several legitimate usernames: `jmartin`, `svc-mon`, `netops` — noted for later.
- The **Host Health** tab let an operator run connectivity probes against arbitrary targets — a user-supplied host handled server-side is immediately worth attacking.

<img width="1218" height="626" alt="Host Health probe feature" src="https://github.com/user-attachments/assets/54aa9ff7-6f56-4fbc-b81d-15c8281298a9" />

---

## 4. Remote Code Execution

Testing the probe confirmed it wrapped a `ping` command and returned raw output. Chaining an `id` command onto the target field was blocked by server-side validation.

<img width="1260" height="673" alt="blocked command injection attempt" src="https://github.com/user-attachments/assets/7969b591-a506-4b31-90a9-8034e1412787" />

Swapping the usual injection characters for a URL-encoded newline bypassed the blocklist:

```
127.0.0.1%0aid
```

This cleared validation cleanly, and the `id` output came back in the response — confirmed RCE.

<img width="1241" height="688" alt="confirmed RCE via id command" src="https://github.com/user-attachments/assets/63efa739-5411-4cd6-b58c-2ec0252f26e4" />

### Credential discovery

Before committing to a full reverse shell, I used the injection point to explore the filesystem and found `secret.config`. Catting it out revealed plaintext credentials.

<img width="1239" height="641" alt="secret.config plaintext credentials" src="https://github.com/user-attachments/assets/229dda16-be32-44cf-a42c-9de6fc7a71d0" />

---

## 5. Foothold — SSH

Port 22 was the only other exposed surface, so I tried the recovered credentials there. They worked.

<img width="441" height="128" alt="SSH login success" src="https://github.com/user-attachments/assets/fc9edc2e-07ca-4b02-8e11-d64eb5a0fe77" />

After grabbing the user flag, I checked `/etc/passwd` for accounts with shell access, to scope out other potential users worth pivoting to.

<img width="478" height="68" alt="/etc/passwd bash users" src="https://github.com/user-attachments/assets/5af160b4-4625-46d6-b80a-0808e3984fe0" />

---

## 6. Privilege Escalation

The `ubuntu` home directory had nothing useful. Revisiting `sysadmin`'s home directory, I caught a `backups` folder I'd skipped over the first pass. Inside was `infrastructure.kdbx` — encrypted, and quickly identified as a KeePass password database.

<img width="683" height="75" alt="infrastructure.kdbx found in backups" src="https://github.com/user-attachments/assets/3fbffdca-67c1-4604-91a3-027f7295528f" />

### 6.1 Cracking the vault

I pulled the database to my local machine to work on it properly.

- **Attempt 1:** Targeted brute force with a custom AI-generated wordlist built from context clues in the room (hostnames, themes, tech stack). No luck.

<img width="761" height="184" alt="failed targeted brute force" src="https://github.com/user-attachments/assets/0e00f5a2-3070-4ca4-b228-b66a9477911f" />

- **Attempt 2:** Converted the database to a crackable hash and handed it to John the Ripper — this found the master password.

<img width="1158" height="363" alt="John the Ripper cracks master password" src="https://github.com/user-attachments/assets/87d5f7c1-fbf9-4ad0-b745-bf7b448587fe" />

### 6.2 Root credentials

With the cracked master password, I unlocked the vault via the KeePass CLI. Browsing the stored entries turned up root's credentials.

<img width="824" height="167" alt="root credentials in KeePass vault" src="https://github.com/user-attachments/assets/f6bdc359-decf-4780-ae4f-951d70422296" />

---

## 7. Root

Switched users to root with the recovered credentials and read out `root.txt`. End of challenge.

<img width="323" height="86" alt="root flag" src="https://github.com/user-attachments/assets/ca9ac2e2-4b00-47d2-8909-85dabb4e900d" />

---

## Key Takeaways

- **Authentication bypass:** No input sanitization on the login form allowed a trivial SQLi payload to bypass auth entirely.
- **Command injection filtering:** Blocklist-based validation on the ping utility was defeated with a simple `%0a` newline encoding — allowlisting or proper shell argument handling would have prevented it.
- **Secrets management:** Plaintext credentials in `secret.config` gave a direct path to SSH access.
- **Backup hygiene:** An unprotected `.kdbx` backup in a home directory, secured only by a crackable master password, was the difference between a user shell and root.
