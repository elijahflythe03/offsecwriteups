# TryHackMe - Monitor Write-Up

I kick things off with an nmap scan. I've been leaning on verbose mode more often lately so I can watch results come in live instead of waiting on the whole scan to finish before seeing anything.

<img width="905" height="557" alt="image" src="https://github.com/user-attachments/assets/342ff9cf-fde0-42c2-9c4e-4904856e143b" />


The scan turns up two open ports: 22, running OpenSSH, and 5050, running a Werkzeug/Python web app. Since there's nothing to do with SSH without credentials yet, I head over to the web app first.

<img width="1358" height="1012" alt="image" src="https://github.com/user-attachments/assets/6122eeab-4f99-425d-b940-c508bd050800" />


It's a completely static landing page — no buttons, forms, or links to click through. Nothing on the surface to interact with, so it's time to dig under the hood instead.

Using ffuf to brute-force directories against the webroot, I turn up a hidden `/internal` path that isn't linked anywhere on the page.

<img width="829" height="121" alt="image" src="https://github.com/user-attachments/assets/780d7484-ea76-4671-adbb-693d9e8ce505" />


Navigating to `/internal` drops me straight into an authentication portal — an operator login screen gating whatever lives behind it. Let's see what's back there.

<img width="485" height="420" alt="image" src="https://github.com/user-attachments/assets/1517a6d6-2544-440a-9045-0ee8dc4b2a0d" />


Since everything past this point is authenticated, before touching the login form itself I want to know what else exists behind it. I run a second directory scan scoped to `/internal` and turn up three more endpoints: `health`, `logout`, and `dashboard`.

<img width="1054" height="479" alt="image" src="https://github.com/user-attachments/assets/1444fc48-c7de-44a9-8be9-d9096d0b4aed" />


With the app mapped out, I start manually testing the login form for injection points — throwing a handful of payloads at the username and password fields to see how the backend reacts. It doesn't take long before the simplest possible SQL injection payload gets a response worth looking at.

<img width="1236" height="405" alt="image" src="https://github.com/user-attachments/assets/fff36e2a-2415-4596-9f41-0dfb404ffabb" />


The login request comes back with a redirect and a fresh session cookie — the auth check got bypassed outright. First thing I do is decode the JWT sitting in that cookie to see what access it actually grants.

<img width="1308" height="421" alt="image" src="https://github.com/user-attachments/assets/c9b36131-a9c2-4ff9-bfb3-88f9234e1421" />


The decoded token confirms it: I'm holding a valid session as a NOC operator.

<img width="1561" height="907" alt="image" src="https://github.com/user-attachments/assets/f7756d34-1d44-4668-96c5-00f1d063638c" />


Poking around the dashboard, the audit log leaks a handful of legitimate usernames — `jmartin`, `svc-mon`, and `netops` — worth keeping in mind for later. More interesting is the Host Health tab, which lets an operator run connectivity probes against arbitrary targets. Any feature that takes a user-supplied host and does something with it on the backend is immediately worth attacking.

<img width="1218" height="626" alt="image" src="https://github.com/user-attachments/assets/54aa9ff7-6f56-4fbc-b81d-15c8281298a9" />


Testing the probe out, it clearly wraps a `ping` command and returns the raw output. I want to know if that's all it does, so I try chaining an `id` command onto the target field — server-side validation blocks it outright.

<img width="1260" height="673" alt="image" src="https://github.com/user-attachments/assets/7969b591-a506-4b31-90a9-8034e1412787" />


To get around the validator blocking the `id` command, I swap out the usual injection characters for a URL-encoded newline instead — the `%0a` bypasses whatever character blocklist is doing the filtering:

127.0.0.1%0aid

This bypasses the validation cleanly and the `id` output comes back in the response — confirmed remote code execution. From here I want to pull whatever information I can off the box through this injection point before committing to a full reverse shell.

<img width="1241" height="688" alt="image" src="https://github.com/user-attachments/assets/63efa739-5411-4cd6-b58c-2ec0252f26e4" />


Poking around the filesystem through the injection, I find a file called `secret.config`. Catting it out with the same RCE turns up plaintext credentials sitting right there in the config.

<img width="1239" height="641" alt="image" src="https://github.com/user-attachments/assets/229dda16-be32-44cf-a42c-9de6fc7a71d0" />


Port 22 is the only other exposed attack surface on the box, so I try authenticating over SSH with the credentials I just pulled. They work — I'm in.

<img width="441" height="128" alt="image" src="https://github.com/user-attachments/assets/fc9edc2e-07ca-4b02-8e11-d64eb5a0fe77" />


After grabbing the user flag, I want to confirm what other accounts on the box actually have shell access, so I check `/etc/passwd` for anything running `bash`.

<img width="478" height="68" alt="image" src="https://github.com/user-attachments/assets/5af160b4-4625-46d6-b80a-0808e3984fe0" />


Digging through the `ubuntu` home directory doesn't turn up much of anything useful. I step back and revisit the `sysadmin` home directory instead, and realize I'd skipped over a `backups` folder the first time through. Inside is an `infrastructure.kdbx` file — but the contents are encrypted when I try to open it. A quick lookup confirms `.kdbx` is a KeePass password database file.

<img width="683" height="75" alt="image" src="https://github.com/user-attachments/assets/3fbffdca-67c1-4604-91a3-027f7295528f" />


I pull the database back to my local machine so I can work on cracking it properly.

First attempt is a targeted brute force using a custom, AI-generated wordlist built off context clues from the room — hostnames, themes, technology in play. No luck.

<img width="761" height="184" alt="image" src="https://github.com/user-attachments/assets/0e00f5a2-3070-4ca4-b228-b66a9477911f" />


Next I convert the database to a crackable hash and hand it off to John the Ripper instead — and this time it finds the master password.

<img width="1158" height="363" alt="image" src="https://github.com/user-attachments/assets/87d5f7c1-fbf9-4ad0-b745-bf7b448587fe" />


With the cracked master password, I unlock the vault through the KeePass CLI, and browsing the stored entries turns up root's credentials.

<img width="824" height="167" alt="image" src="https://github.com/user-attachments/assets/f6bdc359-decf-4780-ae4f-951d70422296" />


Now I switch users to root with those credentials and cat out root.txt. End of challenge.

<img width="323" height="86" alt="image" src="https://github.com/user-attachments/assets/ca9ac2e2-4b00-47d2-8909-85dabb4e900d" />

