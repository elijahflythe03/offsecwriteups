# Silver - TryHackMe Writeup

**Platform:** TryHackMe
**Difficulty:** Medium
**Tags:** Web, IDOR, CVE-2024-36042, Auth Bypass, Privilege Escalation

---

## Recon

First thing I did was add the target IP `10.67.149.182` to `/etc/hosts` as `silver.thm`, then pinged it to confirm connectivity before doing anything else.

<img width="624" height="420" alt="image" src="https://github.com/user-attachments/assets/97ae3f6d-8136-4b4d-b40f-dcfffcad5a93" />


With connectivity confirmed, I kicked off an nmap scan for network recon.

<img width="624" height="379" alt="image" src="https://github.com/user-attachments/assets/059982c3-1a1d-43da-986d-5552a78488a1" />


Scan wasn't the most comprehensive, but it gave us what we needed. Three open ports:

- **22/tcp** - OpenSSH 8.9p1 (Ubuntu)
- **80/tcp** - nginx 1.18.0
- **8080/tcp** - http-proxy (unrecognized service)

The port 8080 service returning 404s and some unusual fingerprint data was worth noting for later. I started browsing port 80 and found a contact form that leaked a potential username and a reference to something called **Silverpeas**. I noted the username and flagged Silverpeas for further investigation.

---

## Directory Enumeration

I ran gobuster against port 80 using the `dirb/big.txt` wordlist with 50 threads.

<img width="624" height="256" alt="image" src="https://github.com/user-attachments/assets/a0c37de1-a9af-4033-8a91-2953e32d7262" />


Only `/assets` and `/images` came back with 301 redirects, nothing that interesting. Still, I kept poking around manually and confirmed that `/silverpeas` exists as a directory. It returns unauthorized on port 80, but port 8080 has a redirect that makes it accessible. That's where I headed next.

---

## Authentication Bypass - CVE-2024-36042

Browsing to port 8080 brought up a `defaultLogin.jsp` auth portal for Silverpeas. I looked it up and found a documented auth bypass.

<img width="975" height="199" alt="image" src="https://github.com/user-attachments/assets/b0e59c69-1576-48e0-b834-0c57b1af81a7" />


CVE-2024-36042 affects Silverpeas before version 6.3.5. The vulnerability lives in the `AuthenticationServlet` and allows unauthenticated access by simply omitting the `Password` field from the POST request, sometimes granting superadmin-level access. Simple and brutal.

I loaded up Burp Suite to intercept the login request.

<img width="611" height="397" alt="image" src="https://github.com/user-attachments/assets/4bc3a45d-881b-42f5-b5c0-71f7feb03673" />

I entered the valid username discovered earlier with a dummy password, then intercepted the POST request through Burp Proxy.

<img width="624" height="248" alt="image" src="https://github.com/user-attachments/assets/62f4c548-37f6-4bb9-a631-4edafa467aa5" />


The intercepted request goes to `/silverpeas/AuthenticationServlet` on port 8080. The body contains `Login=scr1ptkiddy&Password=3333333333&DomainId=0&X-STKN=...`. To exploit the bypass, I deleted the value of the `Password` parameter, leaving it empty (`Password=`), and forwarded the request. That was it.

<img width="624" height="405" alt="image" src="https://github.com/user-attachments/assets/cb436e03-8552-43e8-b370-acee635e0812" />


We're in as `scr1ptkiddy`. The landing page also shows a banner for **1 unread notification** which I made note of for later.

---

## Enumeration Inside Silverpeas

With access to the portal, I browsed to the **Directory** section to see what users were registered.

<img width="624" height="250" alt="image" src="https://github.com/user-attachments/assets/2013cd49-2ed4-4188-a33c-5ebb808c0415" />


Three users: `Manager Manager`, `scr1ptkiddy scr1ptkiddy`, and `Administrateur`. The admin account lists the email `silveradmin@localhost`. I now have a confirmed admin username to work with.

---

## IDOR - Accessing Admin Notifications

Going back to that unread notification, I clicked into it and noticed the URL had a parameter with a very low ID value: `ID=5`. Low entropy like that is a dead giveaway for an Insecure Direct Object Reference vulnerability. I started iterating through adjacent IDs manually.

Changing the ID to `6` pulled up the admin's notification inbox.

![Admin notification containing SSH credentials via IDOR](media/image9.png)

The admin left SSH credentials in a notification to another user. The message reads: `Username: tim` and includes the password in plaintext. ID=3 had another message but nothing useful. Sequential IDs really are fun to play with.

---

## Initial Access - SSH as Tim

I SSH'd into Tim's account using the credentials pulled from the IDOR.

<img width="624" height="231" alt="image" src="https://github.com/user-attachments/assets/1854cd8c-4049-4e36-87d7-151d97efaa7b" />


User flag was low-hanging fruit sitting right in `~/user.txt`. Now time to work on priv esc.

<img width="975" height="130" alt="image" src="https://github.com/user-attachments/assets/e61b571d-73a1-48fc-9ff4-fe11ed286422" />


---

## Privilege Escalation

### Transferring LinPEAS

I spun up a Python HTTP server on port 8081 from my local machine's `~/Downloads` directory where LinPEAS was stored.

<img width="524" height="87" alt="image" src="https://github.com/user-attachments/assets/bb535a1a-2b90-4b19-9c21-459b8256ae93" />

On the target, I used `wget` from `/tmp` to pull the script over from my attacking machine at `192.168.206.16`.
<img width="624" height="402" alt="image" src="https://github.com/user-attachments/assets/dc043fa1-3d30-49bb-8cb7-1044de234608" />


After downloading, I changed the permissions to make it executable and ran it with `-s 2>/dev/null` to suppress stderr and keep the output clean, only showing useful findings.
<img width="624" height="333" alt="image" src="https://github.com/user-attachments/assets/5c133ce4-80f2-4f46-af38-39047117e8ee" />

LinPEAS kept failing partway through for some reason, so I moved on to manual enumeration. I checked `/etc/passwd` for other users on the box and found **tyler**. That gave me a new target for lateral movement.

### Credential Discovery in Auth Logs

I checked the auth logs for anything useful.

![Auth logs showing tyler's activity and docker commands](media/image14.png)

The logs showed tyler running a `docker run` command with `-e POSTGRES_PASSWORD=Zd_zx7N823/` passed as an environment variable in plaintext. That's a credential exposure straight out of the sudo history.

<img width="624" height="299" alt="image" src="https://github.com/user-attachments/assets/7c1f7d85-949b-4ffd-b223-83b043ab767a" />


People reuse passwords. I tried logging into tyler's account with that postgres password.

<img width="975" height="75" alt="image" src="https://github.com/user-attachments/assets/00f37819-d01f-46dc-8487-d5ac6c32af57" />

---

### Lateral Movement to Tyler

I switched users from tim to tyler using the password pulled from the docker command in the auth logs.

<img width="547" height="123" alt="image" src="https://github.com/user-attachments/assets/ce72bc6e-ddb9-412c-91b2-91584298f742" />


It worked. From there, I ran `sudo -l` to check what commands tyler could run as root.

<img width="624" height="99" alt="image" src="https://github.com/user-attachments/assets/e2fe09bb-46ac-45a4-b1d9-8249e706e7e9" />


Tyler had `(ALL : ALL) ALL`, meaning full sudo access. I ran `sudo cat /root/root.txt` and grabbed the root flag.

---

## Summary

This box chained together a few interesting techniques. The Silverpeas auth bypass via CVE-2024-36042 was the entry point, and the IDOR on the notification system handed over SSH credentials without needing to brute force anything. Post-exploitation was straightforward: LinPEAS assisted with enumeration, auth logs leaked a postgres password in plaintext, password reuse got us to tyler, and tyler's unrestricted sudo access closed it out.

| Step | Technique |
|------|-----------|
| Recon | nmap, gobuster, manual browsing |
| Initial foothold | CVE-2024-36042 auth bypass (Silverpeas) |
| Credential access | IDOR on notification endpoint (ID enumeration) |
| SSH access | Credentials leaked via IDOR |
| Privilege escalation | Plaintext creds in auth logs, password reuse, sudo -l |
