# TryHackMe: Heart Write-up

## Initial Setup

Doesn't really give us much, but let's go ahead and create a host in our `/etc/hosts` file for convenience.

<img width="975" height="834" alt="image" src="https://github.com/user-attachments/assets/a5f74ece-2eed-490f-a108-82a790ff697e" />


---

## Nmap Scan

Next we run our standard network scan on our target.

<img width="975" height="704" alt="image" src="https://github.com/user-attachments/assets/53bdab1f-bf09-4cc1-adaa-9b70755e94f9" />


Interesting results, random RTSP request? UPnP?

I go down a rabbit hole on potential protocol level vulnerabilities but it leads nowhere, next I'm going to test for hidden paths, I try `robots.txt` and it's a hit.

<img width="975" height="333" alt="image" src="https://github.com/user-attachments/assets/9d2327bc-eaca-4fac-a234-a24a1ad85e2c" />



---

## Following the Breadcrumbs

I follow the clue and it leads me here, I'm also gonna keep note of the comment, might be useful, could be credentials?

<img width="975" height="489" alt="image" src="https://github.com/user-attachments/assets/a1ff9aa5-4fab-4e76-8f0c-bd8d5d0a9030" />

---

## Directory Fuzzing

From here I attempt to fuzz for more hidden paths, and we get some insightful info returned.

<img width="975" height="444" alt="image" src="https://github.com/user-attachments/assets/1a87ac0b-717f-431f-9878-2ab8de7ebe22" />


---

## Admin Portal Discovery

I head to `/administrator` and would you look at that, we have an auth portal. Let's get started.

<img width="975" height="698" alt="image" src="https://github.com/user-attachments/assets/a721b2a7-f2de-4d51-b694-39d0af7647aa" />


---

## Connecting the Dots

From here I tried to connect the dots... earlier in the `robots.txt` file there was a phrase commented out, so I used the username `admin` and then that string for the password... and first try it worked out!!!

<img width="975" height="412" alt="image" src="https://github.com/user-attachments/assets/ab280b02-db23-4fd4-ae1b-83cd8054f64e" />


Challenge completed!

---

## Summary

**Key Steps:**
1. Added target to `/etc/hosts`
2. Ran nmap scan, discovered port 5000 running Flask/Werkzeug
3. Checked `robots.txt` - found hidden directory and potential credential
4. Fuzzed `/cupids_secret_vault/` directory
5. Found `/administrator` login portal
6. Used credentials from robots.txt comment: `admin:cupid_arrow_2026!!!`
7. Successfully authenticated and completed the challenge

**Tools Used:**
- Nmap - network reconnaissance
- ffuf/Gobuster - directory fuzzing
- Firefox - manual web testing

**Lesson Learned:**
Always check `robots.txt` for hidden paths and pay attention to comments - they often contain valuable hints or credentials!
