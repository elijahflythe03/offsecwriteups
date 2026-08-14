# Lookback THM

## Recon

Ran an Nmap scan, identified the target hostname, and added it to `/etc/hosts`.

<img width="624" height="336" alt="image" src="https://github.com/user-attachments/assets/a045a793-d80c-400b-9979-50ab55eb3a9a" />


Performed directory enumeration against the web host, yielding the following results.

<img width="624" height="311" alt="image" src="https://github.com/user-attachments/assets/7e2f059f-e5e4-44e1-9921-425ddb76d4ad" />


The `/Test` directory returns an authentication prompt.

<img width="624" height="377" alt="image" src="https://github.com/user-attachments/assets/5c7fad6c-7d7c-461d-b5f4-3b9cae3ed542" />


Ran Nikto against the web app for additional context, which surfaced a set of default credentials. These failed against the primary portal, so I attempted them against the Test directory's auth prompt instead.

<img width="624" height="249" alt="image" src="https://github.com/user-attachments/assets/39d3ee2d-f000-4c95-a206-7787abf54e13" />


## Initial Foothold

Authentication succeeds, granting the first service-user flag.

<img width="624" height="254" alt="image" src="https://github.com/user-attachments/assets/566141be-8b29-4ca6-a234-cde3801e804f" />


The authenticated page exposes a text box that executes commands. After testing input syntax, the working injection pattern is:

');command;#

<img width="624" height="290" alt="image" src="https://github.com/user-attachments/assets/5b263439-c30e-4cdb-af5f-bba75ecdd00e" />


Using this syntax, I sent a base64-encoded payload and obtained a shell as `inetsrv`.

<img width="579" height="156" alt="image" src="https://github.com/user-attachments/assets/cf5470b5-2e5d-4495-ba08-2609794ccf97" />


## Windows System Enumeration

With shell access established, began Windows host enumeration.

Enumerating `Users` reveals two accounts: `Admin` and `Dev`. Under `Dev\Desktop`, I locate `user.txt`.

<img width="618" height="608" alt="image" src="https://github.com/user-attachments/assets/5d71c300-decb-466d-b365-d92dbd5ccbe4" />


Reviewing `TODO.txt` for additional context reveals MS Exchange is installed with a pending security update `[TO BE DONE]` — a likely vulnerability. Searched Exploit-DB via `searchsploit` for related MS Exchange CVEs and also recovered several user email addresses from the file.

<img width="528" height="359" alt="image" src="https://github.com/user-attachments/assets/0334fae2-d9d6-4b5a-b442-8108de8612d7" />


## Exploitation

The `searchsploit` query returns numerous candidate exploits, so I pivot to Metasploit to select a module.

Modules 16 and 8 appear to best match the target's version/configuration; module 16 is tried first.

<img width="975" height="455" alt="image" src="https://github.com/user-attachments/assets/3bddc6e2-3f56-44d8-a633-15ce1f676b97" />


Exploitation succeeds, dropping into a shell. Enumerated current privileges before continuing.

<img width="624" height="304" alt="image" src="https://github.com/user-attachments/assets/23143b58-d47d-4138-b879-9f7760782bec" />


The resulting privilege level provides significantly more access than the initial foothold shell.

<img width="624" height="243" alt="image" src="https://github.com/user-attachments/assets/bc9f425d-0231-4c9d-8690-e99e1d51ebb8" />


## Flag & Privilege Escalation

Located the next flag via a system-wide file search and read its contents using the elevated privileges obtained above — challenge complete.

<img width="352" height="69" alt="image" src="https://github.com/user-attachments/assets/8bdc05d9-576e-4502-8062-b9767353b6b6" />




