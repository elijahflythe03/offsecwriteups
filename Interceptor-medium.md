# MediaHub - Walkthrough

## Reconnaissance

The engagement started with an `nmap` service/version scan to map the target's exposed attack surface.

<img width="804" height="234" alt="image" src="https://github.com/user-attachments/assets/dea05358-5bfc-431b-986a-f9d4944cbd2a" />

The scan pointed to a web application on port 80. The landing page was minimal but exposed a `login.php` authentication portal. While reviewing it, I kicked off directory fuzzing in the background. Inspecting the portal's source revealed that all login attempts are submitted to a backend API endpoint:

<img width="382" height="93" alt="image" src="https://github.com/user-attachments/assets/fddf9874-2ee8-468b-ba48-8dc231b36f23" />

---

## Enumeration - Directory Fuzzing

The background `ffuf` scan completed and surfaced several additional paths.

<img width="879" height="363" alt="image" src="https://github.com/user-attachments/assets/33187d28-9ecc-4288-b543-6c6b4ed811ba" />

Working through the discovered files, a `.bak` backup file contained a note that proved to be the key to authentication - a reminder documenting the admin password format.

<img width="682" height="308" alt="image" src="https://github.com/user-attachments/assets/058a4543-775f-4008-9697-fd03c16eabe8" />

---

## Credential Brute Force

The note confirmed a valid account, `admin@mediahub.thm`, but disclosed only the password *format* - the string `MediaHub` followed by a year - rather than the full value. Instead of testing each year by hand, I wrote a Python script to iterate the candidate set.

Burp Intruder initially hit a wall: after roughly five attempts the endpoint blocked further tries. The rate limiting was keyed to the `PHPSESSID` cookie, so the fix was to force a fresh session on every request - the script rotates the session token per attempt, sidestepping the limit entirely.

<img width="795" height="729" alt="image" src="https://github.com/user-attachments/assets/a5297aa2-56e4-4ad4-8402-451476a5a677" />

After a few tweaks, the script ran cleanly and recovered valid credentials.

<img width="1218" height="428" alt="image" src="https://github.com/user-attachments/assets/afe192a9-e2f8-4dc5-85e4-ccbecc18771b" />

---

## Two-Factor Authentication Bypass (Logic Flaw)

Authentication succeeded but dropped me at a 2FA / OTP challenge. I opened Burp to inspect the flow and confirmed the code is verified at verify_otp.php.

<img width="686" height="353" alt="image" src="https://github.com/user-attachments/assets/4b17c4ed-b7e5-4ca6-a2c1-21fe68ec5eb1" />

After ruling out a few approaches - injection attempts, and weighing a brute force of the six-digit space - I shifted to probing the verification logic itself.

<img width="1285" height="401" alt="image" src="https://github.com/user-attachments/assets/824f49d7-9857-41cb-a066-d38d9db8aa10" />

The verification came down to a single boolean (verified: true / false). The bypass was in how the request was framed: resubmitting the original multipart form-data failed, but re-sending the request with an equivalent JSON body let me force the verified state to `true` and clear the gate.

<img width="1125" height="128" alt="image" src="https://github.com/user-attachments/assets/a395475f-d3c6-48a8-a47b-28753835cf60" />

---

## Command Injection / SSRF - Flag Capture

The bypass landed me on `dashboard.php`, which exposed two notable features: a profile-picture upload and a URL "import feed" fetcher. The fetcher - server-side retrieval of a user-supplied URL - immediately read as a candidate for SSRF or command execution.

<img width="1372" height="560" alt="image" src="https://github.com/user-attachments/assets/53e52590-080c-400c-ae81-fe927e0c359e" />

The endpoint was wrapped in layered guardrails, each of which fell in turn:

- a **client-side filter** stripping `;`, `&`, and `|` - bypassed by sending requests directly to the API instead of through the browser
- a **URL validator** rejecting non-HTTP schemes (`file://` returned `Invalid URL`)
- an **SSRF filter** blocking private hosts (`127.0.0.1` returned `Private network access blocked`)

The filter never accounted for command substitution or `${IFS}`, and the private-host check only inspected the parsed hostname. Placing the injected command in the **host position** - with `${IFS}` substituting for spaces to survive URL parsing - passed every check, and `curl`'s failure to resolve the command's output as a hostname leaked the file contents back in the response:

<img width="1270" height="323" alt="image" src="https://github.com/user-attachments/assets/7788358f-47c8-41fa-816b-524be693667a" />

This returned the contents of `/var/www/user.txt` in the resolver error, capturing the flag.
