# TryHackMe: Takeover Write-up

## Initial Setup

First things first, I needed to add the target to my `/etc/hosts` file:
```bash
sudo nano /etc/hosts
```

Added `futurevera` to `/etc/hosts` and then checked my connectivity to make sure everything was working.

<img width="886" height="350" alt="image" src="https://github.com/user-attachments/assets/bc46fc63-7f2f-4780-b85e-313da6186956" />

---

## Nmap Scan

Started with an nmap scan that uses the nmap scripting engine as well as the aggressive scan which does further enumeration into the target. This is what I found:

<img width="975" height="771" alt="image" src="https://github.com/user-attachments/assets/1ce70606-ae2a-4680-8f23-c25781b8ccc5" />

---

## Manual Web Inspection

After this I went to Firefox to manually observe the website and get a lay of the land. It's really just a one pager, nothing on the surface.

<img width="975" height="611" alt="image" src="https://github.com/user-attachments/assets/be7994b9-159f-46e2-b27c-7225f6034c39" />

---

## Directory Enumeration with Gobuster

Next I started my deep enumeration with Gobuster, running directory brute forcing to find any possible paths.

<img width="975" height="264" alt="image" src="https://github.com/user-attachments/assets/0cbeab6d-2838-4873-bb4a-9ead0768c107" />

**Scan one:** No luck, but I remember that in the nmap scan it redirects from 80 to 443, so I just dropped it back to port 80 and got some results for scan two:

<img width="975" height="299" alt="image" src="https://github.com/user-attachments/assets/3ef152a2-cf82-4a12-8bbe-260334543d3e" />

It yielded these results, not very helpful because I don't have proper access to view the page indicated by the 403 response.

---

## Virtual Host Scanning

Next I wanted to scan for vhosts, for further information:

<img width="975" height="308" alt="image" src="https://github.com/user-attachments/assets/ebb72d39-15b0-4211-a4f7-4e638239ad15" />

Bet, finally a hit! Next I'm going to add this to my `/etc/hosts` file to visit the subdomain.

---

## Investigating the Subdomain

Upon visiting the new page it looks the same, so I'm just gonna look around the internals and see what I can find.

<img width="975" height="757" alt="image" src="https://github.com/user-attachments/assets/c01e1c70-8a62-4087-8eeb-45c0d7d0099f" />

After checking certificates and doing technical recon it didn't really return anything fruitful, so I started manually brute forcing subdomains based on the TryHackMe description. I had Claude generate a wordlist based on the description and then used my human judgement to test the most likely ones.

<img width="975" height="587" alt="image" src="https://github.com/user-attachments/assets/4bad8ffb-edbd-412e-8b8d-21938137f9b8" />

---

## Manual Subdomain Testing

**First up, blog** - beginner's luck I guess?

<img width="975" height="669" alt="image" src="https://github.com/user-attachments/assets/28108bbc-b60b-4165-af09-9e7043061036" />

Checked through the certificate and didn't really find anything so we move on to the next in line - **support**.

<img width="975" height="553" alt="image" src="https://github.com/user-attachments/assets/7b279fbf-805e-4422-a512-0e2d7a4abafe" />

Upon checking the certificate for this path I find another potential subdomain, let's see!

<img width="975" height="583" alt="image" src="https://github.com/user-attachments/assets/4f60ef6f-4003-4a74-b215-39aeda483f38" />

---

## Finding the Flag

I eagerly add it to `/etc/hosts` so that I can view the content.

When visiting the subdomain the flag is revealed!!!

<img width="975" height="469" alt="image" src="https://github.com/user-attachments/assets/0be290c1-a1c1-45bd-bc3f-894303349eb1" />

---

## Summary

**Tools Used:**
- Nmap - for initial reconnaissance
- Gobuster - for directory and vhost enumeration
- Firefox - for manual inspection and certificate analysis
- Claude AI - for generating context-aware wordlists

**Key Takeaways:**
- Always check SSL certificates - they can leak information 
- When automated scans don't work, switch to manual testing with contextual clues
- Port 80 and 443 can behave differently - test both

Challenge completed!
