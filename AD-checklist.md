# Active Directory Pentest / Lab Checklist

A working checklist for authorized AD labs (TryHackMe, HackTheBox, GOAD, similar sandboxed ranges). Organized by attack-chain phase. Not every item applies to every box — treat it as a menu, not a script. Confirm tool flag syntax with `-h`/`--help` before relying on it; syntax drifts across versions (Impacket, BloodHound/SharpHound, NetExec/CrackMapExec, Rubeus, Mimikatz especially).

---

## Phase 0 — Port & Service Discovery

Full TCP + UDP scan first, then service/version detection on what's open. A DC has a distinctive port signature — seeing several of these together is a strong DC indicator.

### Core AD/DC ports to check specifically

| Port | Protocol | Service | Why it matters |
|---|---|---|---|
| 53 | TCP/UDP | DNS | AD-integrated DNS; zone transfer attempts (`dig axfr @dc domain.tld`), SRV record enumeration (`_ldap._tcp`, `_kerberos._tcp`) reveals DC roles |
| 88 | TCP/UDP | Kerberos | Pre-auth username enum (Kerbrute), AS-REQ/TGS-REQ attacks (AS-REP roast, Kerberoast), clock-skew sensitivity |
| 135 | TCP | RPC Endpoint Mapper | `rpcclient`, `impacket` tools resolve dynamic RPC ports through this |
| 137/138 | UDP | NetBIOS Name/Datagram | Legacy name resolution, `nbtscan` |
| 139 | TCP | NetBIOS Session (SMB) | Older SMB access path |
| 389 | TCP/UDP | LDAP | Unauthenticated/anonymous bind checks, directory enumeration |
| 445 | TCP | SMB | Share enum, null/guest sessions, `NTDS.dit` access if privileged |
| 464 | TCP/UDP | kpasswd | Kerberos password change service |
| 593 | TCP | RPC over HTTP | Alternate RPC transport |
| 636 | TCP | LDAPS | Encrypted LDAP — same enum as 389 but check certs |
| 3268 | TCP | Global Catalog | Forest-wide search (multi-domain forests) |
| 3269 | TCP | Global Catalog SSL | Encrypted GC |
| 3389 | TCP | RDP | Direct interactive access if creds/hashes work |
| 5985 | TCP | WinRM (HTTP) | Evil-WinRM, PS remoting |
| 5986 | TCP | WinRM (HTTPS) | Same, encrypted |
| 9389 | TCP | AD Web Services (ADWS) | PowerShell AD module remote queries; also abusable via SOAPHound |
| 1433 | TCP | MSSQL | If present — check for `sysadmin`/linked-server abuse, `xp_cmdshell` |
| 25/587 | TCP | SMTP | If Exchange present — user enum via VRFY/RCPT, internal mail relay |
| 80/443 | TCP | HTTP(S) | ADCS web enrollment (`/certsrv`), internal web apps, Exchange OWA |

**Commands:**
```
nmap -p- --min-rate 5000 -oN allports <target>
nmap -sC -sV -p<open-ports> -oN detailed <target>
nmap -sU --top-ports 100 <target>     # don't skip UDP — DNS/Kerberos live here too
```

Add discovered domain name + DC FQDN to `/etc/hosts` immediately — Kerberos is name-sensitive and will misbehave against bare IPs.

---

## Phase 1 — Unauthenticated / External Recon

- [ ] SMB null session check: `smbclient -N -L //<dc>/` or `netexec smb <dc> -u '' -p ''`
- [ ] SMB signing status (relevant later for NTLM relay): `netexec smb <dc> --gen-relay-list out.txt`
- [ ] LDAP anonymous bind: `ldapsearch -x -h <dc> -s base namingcontexts`
- [ ] If anonymous LDAP works — pull domain naming context, then enumerate users/groups/computers:
  ```
  ldapsearch -x -h <dc> -b "DC=domain,DC=tld" "(objectClass=user)"
  ```
- [ ] RPC null session: `rpcclient -U "" -N <dc>` → try `enumdomusers`, `enumdomgroups`, `querydominfo`
- [ ] SNMP if UDP/161 open — community string `public` guesses, `snmpwalk`
- [ ] DNS zone transfer attempt: `dig axfr @<dc> <domain>`
- [ ] Check NTP (UDP/123) — needed for Kerberos clock sync if you're time-skewed relative to the DC (`ntpdate`/`rdate` or `sudo ntpdate -u <dc>`)
- [ ] Note: OS/domain functional level clues from `smbclient`/`netexec` banner grabs

**What this phase should tell you:** domain name, DC hostname(s), whether anonymous enum is viable at all, whether SMB signing is disabled anywhere (relay potential), rough account/group inventory if LDAP/RPC null sessions work.

---

## Phase 2 — Username Enumeration

- [ ] Gather any known employee/dept names from prior recon (web content, LinkedIn-style OSINT, earlier-stage loot)
- [ ] Generate candidate usernames against common conventions with **username-anarchy** (or manually): `first.last`, `flast`, `firstl`, `first`, `last`
- [ ] Validate candidates via Kerberos pre-auth with **Kerbrute** (`kerbrute userenum`) — relies on the KDC returning `KRB5KDC_ERR_PREAUTH_REQUIRED` (valid user) vs `KDC_ERR_C_PRINCIPAL_UNKNOWN` (invalid) without a full auth attempt or lockout risk
- [ ] Cross-check with RPC/LDAP enum results from Phase 1 if available
- [ ] If a web app or SMTP service is present, check for separate username-disclosure vectors (login error differences, VRFY/RCPT TO)

---

## Phase 3 — Credential Attacks (No Valid Creds Yet)

- [ ] **AS-REP Roasting** — targets accounts with Kerberos pre-auth disabled; no valid creds required to identify candidates:
  ```
  GetNPUsers.py domain/ -usersfile users.txt -no-pass -format hashcat
  ```
- [ ] **Password spraying** — **check lockout policy first**, always:
  ```
  netexec smb <dc> -u users.txt -p 'SeasonYear!' --continue-on-success
  ```
  One password per sweep, spaced out, to stay under lockout thresholds. Common seasonal/company-name patterns worth trying in a lab context.
- [ ] Credential reuse from any earlier-compromised host/service in the same engagement — check every password you've already found against the new user list
- [ ] Default/vendor creds if any non-Windows appliance is in scope

**Don't conflate:** AS-REP roasting needs no creds and targets a specific misconfig (pre-auth disabled); Kerberoasting (Phase 5) needs *any* valid domain cred and targets SPN-bearing accounts. Different prerequisites, different outputs.

---

## Phase 4 — Foothold Confirmation & Low-Priv Enumeration

Once you have one valid credential (password, hash, or ticket):

- [ ] Confirm access level: `netexec smb <dc> -u user -p pass` (look for `(Pwn3d!)` tag = local admin)
- [ ] Enumerate accessible SMB shares: `netexec smb <dc> -u user -p pass --shares` or `smbclient -L`
- [ ] Actually browse non-default shares — configs, scripts, notes, backups often live here
- [ ] Check WinRM group membership / access: `netexec winrm <dc> -u user -p pass`
- [ ] If WinRM works, connect: `evil-winrm -i <dc> -u user -p pass`
- [ ] Once shelled: check local files for creds (Desktop, Documents, `C:\Scripts`, `C:\inetpub`, IIS web.configs, unattended install files, `C:\Windows\Panther\Unattend.xml`)
- [ ] Check current user's local privileges: `whoami /priv`, `whoami /groups`
- [ ] Check for saved credentials: `cmdkey /list`, browser-saved creds if GUI access

---

## Phase 5 — Authenticated Enumeration (BloodHound)

- [ ] Collect with **bloodhound-python** (or SharpHound if you have a foothold on a domain-joined box):
  ```
  bloodhound-python -u <user> -p “password” -d k2.thm -v — zip -c All -dc K2Server.k2.thm -ns 10.10.61.132
  ```
- [ ] Confirm SharpHound collector version matches your BloodHound UI's expected schema (BloodHound Legacy vs BloodHound CE use different data formats — mismatches fail silently or partially)
- [ ] In the UI, check for the current user/group's **Outbound Object Control**:
  - `GenericAll` — full control (reset password, modify most attributes)
  - `GenericWrite` — write most attributes (can be abused for targeted Kerberoasting by setting an SPN, or Shadow Credentials)
  - `WriteDacl` — grant yourself further rights
  - `WriteOwner` — take ownership, then grant rights
  - `ForceChangePassword` — reset password without knowing the old one, narrower than GenericAll
  - `AddSelf` / `AddMember` — group membership manipulation
  - `ReadGMSAPassword` — read a Group Managed Service Account's password directly
  - `AllowedToDelegate` / RBCD-related edges — delegation abuse setup
- [ ] Check **Kerberoastable users** (accounts with SPNs) and **AS-REP roastable users** panels directly in BloodHound
- [ ] Check for `Domain Admins` / `Enterprise Admins` membership paths — BloodHound's shortest-path queries are built for exactly this
- [ ] Check computer objects for **Unconstrained Delegation** flags — high-value targets, since any user who authenticates to them leaves a usable TGT in memory
- [ ] Note any cross-domain/forest trust edges if the lab spans multiple domains

---

## Phase 6 — Credential Attacks (With Valid Creds)

- [ ] **Kerberoasting** (any valid cred, targets SPN accounts):
  ```
  GetUserSPNs.py domain/user:pass -dc-ip <dc> -request
  ```
  Crack offline with hashcat mode 13100.
- [ ] Abuse whatever specific BloodHound edge you found (Phase 5) — reset a password (`net rpc password` or `pywerview`/`bloodyAD`), add group membership, modify a GPO, etc., matched to the exact edge type
- [ ] If `GenericWrite`/similar on a computer object — consider Shadow Credentials (`Key Credential Link` abuse via `pywhisker`) as an alternative to a password reset
- [ ] Re-run BloodHound collection after any credential/membership change — new access frequently opens new edges

---

## Phase 7 — Lateral Movement

- [ ] **Pass-the-Hash**: `evil-winrm -i target -u user -H <ntlm_hash>` or `psexec.py`/`wmiexec.py -hashes`
- [ ] **Pass-the-Ticket**: convert/inject `.kirbi` (Windows/Rubeus format) or ccache (Linux/Impacket format) as appropriate — format mismatches are a common silent failure point
- [ ] **Overpass-the-Hash**: use an NTLM hash to request a real Kerberos TGT (`getTGT.py` / Rubeus `asktgt`), useful when Kerberos-only auth is enforced
- [ ] Delegation abuse if flagged in BloodHound:
  - Unconstrained delegation — coerce/wait for a privileged authentication, capture the TGT from memory
  - Constrained delegation (with/without protocol transition) — `S4U2Self`/`S4U2Proxy` abuse via Rubeus `s4u` or Impacket `getST.py`
  - RBCD — requires write access to a computer object's `msDS-AllowedToActOnBehalfOfOtherIdentity`, then impersonate via S4U
  (These three are distinct primitives — don't collapse their setup/abuse paths together)

---

## Phase 8 — Domain Privilege Escalation / Domain Dominance

- [ ] **DCSync** — requires `Replicating Directory Changes` + `Replicating Directory Changes All` rights:
  ```
  secretsdump.py domain/user:pass@<dc> -just-dc
  ```
- [ ] **NTDS.dit extraction via local privilege** (if `SeBackupPrivilege`/`SeRestorePrivilege` present on a DC shell) — VSS shadow copy (`diskshadow`) → copy `ntds.dit` + `SYSTEM` hive out of the shadow → `secretsdump.py -system System -ntds ntds.dit local`. Functionally similar end result to DCSync, different mechanism/detection surface.
- [ ] **Golden Ticket** (krbtgt hash) vs **Silver Ticket** (specific service account hash) — confirm which one a given technique produces before using it; scope and detection surface differ
- [ ] Check for **ADCS (Active Directory Certificate Services)** misconfigurations if a CA is present — vulnerable templates, ESC1-ESC8-style issues. Confirm the exact vulnerable condition with Certipy/`certipy find` rather than assuming a CA implies a specific ESC number.
- [ ] Named CVE chains (Zerologon, PrintNightmare, noPac, etc.) — **treat like any CVE claim**: confirm the exact CVE number and whether the target is patched before assuming applicability just because it's a DC
- [ ] Trust abuse if multi-domain/forest — SID history injection, trust key extraction

---

## Cross-Cutting Habits (every phase)

- [ ] Re-check lockout policy before *any* spray, every time — `netexec smb <dc> -u user -p pass --pass-pol`
- [ ] Confirm tool version before relying on flag behavior — Impacket, NetExec/CrackMapExec, BloodHound/SharpHound, Rubeus, Mimikatz all drift across releases
- [ ] Keep a running credential/hash table (user, password/hash, source, confirmed access level) — you will reuse these constantly
- [ ] Re-run BloodHound collection after every privilege change, not just once at the start
- [ ] Distinguish "I'm confident" vs "best recollection, verify" vs "unknown" when reasoning about any tool's expected behavior — same standard applies to you as to me
