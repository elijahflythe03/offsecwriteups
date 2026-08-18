# TryHackMe - Monitor Write-Up

I kick things off with an nmap scan. I've been leaning on verbose mode more often lately so I can watch results come in live instead of waiting on the whole scan to finish before seeing anything.

![alt text](image.png)

The scan turns up two open ports: 22, running OpenSSH, and 5050, running a Werkzeug/Python web app. Since there's nothing to do with SSH without credentials yet, I head over to the web app first.

![alt text](image-2.png)

It's a completely static landing page — no buttons, forms, or links to click through. Nothing on the surface to interact with, so it's time to dig under the hood instead.

Using ffuf to brute-force directories against the webroot, I turn up a hidden `/internal` path that isn't linked anywhere on the page.

![alt text](image-3.png)

Navigating to `/internal` drops me straight into an authentication portal — an operator login screen gating whatever lives behind it. Let's see what's back there.

![alt text](image-4.png)

Since everything past this point is authenticated, before touching the login form itself I want to know what else exists behind it. I run a second directory scan scoped to `/internal` and turn up three more endpoints: `health`, `logout`, and `dashboard`.

![alt text](image-5.png)

With the app mapped out, I start manually testing the login form for injection points — throwing a handful of payloads at the username and password fields to see how the backend reacts. It doesn't take long before the simplest possible SQL injection payload gets a response worth looking at.

![alt text](image-6.png)

The login request comes back with a redirect and a fresh session cookie — the auth check got bypassed outright. First thing I do is decode the JWT sitting in that cookie to see what access it actually grants.

![alt text](image-7.png)

The decoded token confirms it: I'm holding a valid session as a NOC operator.

![alt text](image-8.png)

Poking around the dashboard, the audit log leaks a handful of legitimate usernames — `jmartin`, `svc-mon`, and `netops` — worth keeping in mind for later. More interesting is the Host Health tab, which lets an operator run connectivity probes against arbitrary targets. Any feature that takes a user-supplied host and does something with it on the backend is immediately worth attacking.

![alt text](image-9.png)

Testing the probe out, it clearly wraps a `ping` command and returns the raw output. I want to know if that's all it does, so I try chaining an `id` command onto the target field — server-side validation blocks it outright.

![alt text](image-10.png)

To get around the validator blocking the `id` command, I swap out the usual injection characters for a URL-encoded newline instead — the `%0a` bypasses whatever character blocklist is doing the filtering:

127.0.0.1%0aid

This bypasses the validation cleanly and the `id` output comes back in the response — confirmed remote code execution. From here I want to pull whatever information I can off the box through this injection point before committing to a full reverse shell.

![alt text](image-11.png)

Poking around the filesystem through the injection, I find a file called `secret.config`. Catting it out with the same RCE turns up plaintext credentials sitting right there in the config.

![alt text](image-12.png)

Port 22 is the only other exposed attack surface on the box, so I try authenticating over SSH with the credentials I just pulled. They work — I'm in.

![alt text](image-13.png)

After grabbing the user flag, I want to confirm what other accounts on the box actually have shell access, so I check `/etc/passwd` for anything running `bash`.

![alt text](image-14.png)

Digging through the `ubuntu` home directory doesn't turn up much of anything useful. I step back and revisit the `sysadmin` home directory instead, and realize I'd skipped over a `backups` folder the first time through. Inside is an `infrastructure.kdbx` file — but the contents are encrypted when I try to open it. A quick lookup confirms `.kdbx` is a KeePass password database file.

![alt text](image-15.png)

I pull the database back to my local machine so I can work on cracking it properly.

First attempt is a targeted brute force using a custom, AI-generated wordlist built off context clues from the room — hostnames, themes, technology in play. No luck.

![alt text](image-16.png)

Next I convert the database to a crackable hash and hand it off to John the Ripper instead — and this time it finds the master password.

![alt text](image-17.png)

With the cracked master password, I unlock the vault through the KeePass CLI, and browsing the stored entries turns up root's credentials.

![alt text](image-18.png)

Now I switch users to root with those credentials and cat out root.txt. End of challenge.

![alt text](image-19.png)
