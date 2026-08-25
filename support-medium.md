# support.thm — Walkthrough

---

## Reconnaissance

I started with a verbose network scan and quickly identified a web application on port 80 and SSH on port 22.

<img width="822" height="608" alt="image" src="https://github.com/user-attachments/assets/e8f395ac-11e7-4aec-b448-1607f10633c3" />

The web application turned out to be an employee authentication portal for a "Support Operations Panel." The login page includes a note to contact IT Operations at:

```
help@support.thm
```

— a potential valid account worth keeping in mind.

<img width="1720" height="576" alt="image" src="https://github.com/user-attachments/assets/020afa6c-18e3-40d8-906a-b023fee1a9db" />

Further reconnaissance with Nikto surfaced a couple of hidden routes:

```
/config.php
/info.php
```

<img width="1091" height="171" alt="image" src="https://github.com/user-attachments/assets/993f6b6c-3bba-4b5f-bceb-b32551fdf61a" />

Browsing to `info.php` revealed an extensive PHP configuration dump — system specifications and environment details.

<img width="1054" height="1056" alt="image" src="https://github.com/user-attachments/assets/9bc28f3e-139f-4145-994c-10d418bb98de" />

This information wasn't directly actionable while unauthenticated, so I moved on with the goal of achieving an authentication bypass or surfacing valid credentials.

---

## Credential Brute Force

After testing a number of SQL injection and XSS payloads without triggering any errors or unusual behavior, I turned to Hydra and brute-forced the known account against the `rockyou.txt` wordlist:

```bash
hydra -l help@support.thm -P /usr/share/wordlists/rockyou.txt support.thm http-post-form "/:email=^USER^&password=^PASS^:F=Invalid Credentials"
```

This successfully recovered the account's password.

<img width="906" height="159" alt="image" src="https://github.com/user-attachments/assets/665dcdd4-8588-4eb0-a457-b9ea6181ccf9" />

Logging in landed me in what appeared to be a ticket management system, but the page rendered essentially blank.

<img width="1769" height="448" alt="image" src="https://github.com/user-attachments/assets/6da54dff-005b-45a3-b4d2-adb39563c9ee" />

---

## Parameter Discovery & Failed Injection Attempts

A line-by-line review of the page source didn't turn up anything of interest — the only functional element on the page was a theme selector.

<img width="301" height="241" alt="image" src="https://github.com/user-attachments/assets/0a5a7b30-c2f1-4545-a7fa-43116b599e54" />

Experimenting with the theme selector revealed a parameter in the URL:

```
?skin=
```

— an immediate candidate for injection testing.

<img width="334" height="44" alt="image" src="https://github.com/user-attachments/assets/be22c37e-a202-4c26-bff6-5f362ccab81d" />

Despite hundreds of manual and automated injection attempts against the `skin` parameter, nothing proved fruitful.

---

## Authentication Bypass via Cookie Manipulation

I shifted focus to the only other attack surface identified so far: an `isITUser` cookie holding what looked like an MD5 hash.

```
isITUser=68934a3e9455fa72420237eb05902327
```

Decrypting it revealed the plaintext value:

```
false
```

I computed the corresponding hash for `true`:

```
true  →  b326b5062b2f0e69046810717534cb09
```

Flipping the cookie to this value and resending the request to `/dashboard.php` granted access to an admin panel.

<img width="930" height="312" alt="image" src="https://github.com/user-attachments/assets/961efd20-1e2b-4552-b122-f3d7fb8b1a7b" />

This admin panel unlocked access to `api.php`, a route that had previously returned access-denied errors.

<img width="1388" height="305" alt="image" src="https://github.com/user-attachments/assets/880fdc26-2dec-405a-8115-cf35f4f68612" />

---

## IDOR — Insecure Direct Object Reference

`api.php` turned out to be a reference guide for an internal API. It documented the ability to "query your own profile" via:

```
/user/3
```

This immediately raised the question of whether users `1`, `2`, `4`, `5`, and so on were also reachable.

<img width="1344" height="468" alt="image" src="https://github.com/user-attachments/assets/fac23fea-0045-44f3-b21a-8114f5f3cfd8" />

I first confirmed the endpoint's expected behavior against my own profile before probing for an IDOR.

<img width="1045" height="302" alt="image" src="https://github.com/user-attachments/assets/10251017-0bb0-4bdf-a9b5-aa031f0405c9" />

With the API responding as expected, I began testing adjacent user IDs sequentially:

```
GET /user/1
```

This returned admin account information — a different user entirely.

<img width="1064" height="281" alt="image" src="https://github.com/user-attachments/assets/ac9f18e0-52d0-474f-aa0f-4d0a68797632" />

---

## Local File Inclusion — Credential Disclosure

At this point the objective became finding valid credentials for the admin account, since that's where the first flag was expected to live. Recalling that the earlier Nikto scan flagged a potential `/config` directory, I tested the `skin` parameter for local file inclusion:

```
?skin=../config
```

This disclosed its contents.

<img width="1212" height="480" alt="image" src="https://github.com/user-attachments/assets/a3dd53b5-e625-4263-9de5-83de74a3ac89" />

Using the disclosed credentials, I logged into the admin account and retrieved the first flag.

<img width="1342" height="498" alt="image" src="https://github.com/user-attachments/assets/f2f33e25-6647-41bc-9d00-c6f8f28c6525" />

---

## Command Injection — User Flag

With admin access established, I turned to locating `user.txt`. Inside the admin portal I found new "Date" and "Time" buttons; testing this functionality revealed a parameter driving the underlying request:

```
sys=
```

This functionality was only exposed post-authentication as admin, so I used that access to attempt command injection and exfiltrate `/home/ubuntu/user.txt`.

<img width="804" height="206" alt="image" src="https://github.com/user-attachments/assets/40a16ae6-0814-47db-9670-2cf567c94229" />

After working out the correct injection syntax for the `sys` parameter, the following payload returned the flag:

```
sys=date; cat /home/ubuntu/user.txt
```

<img width="1210" height="539" alt="image" src="https://github.com/user-attachments/assets/5c26654b-cb49-4998-a44e-14df756987ac" />
