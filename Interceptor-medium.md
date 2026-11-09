As always I kick this off with an nmap scan. 

<img width="804" height="234" alt="image" src="https://github.com/user-attachments/assets/dea05358-5bfc-431b-986a-f9d4944cbd2a" />

I head to where the web app is hosted, port 80, and im met with a very basic page, but there is a path to login.php which is an auth portal. while I inspect this I let directory fuzzing run in the background. After looking at the source code for the auth portal I see that all login attempts are being posted to api_login.php.

<img width="382" height="93" alt="image" src="https://github.com/user-attachments/assets/fddf9874-2ee8-468b-ba48-8dc231b36f23" />

In the background my ffuf scan finished, showing us a couple paths 

<img width="879" height="363" alt="image" src="https://github.com/user-attachments/assets/33187d28-9ecc-4288-b543-6c6b4ed811ba" />

Checked everything out, and the .bak file holds a note that is our key to auth 

<img width="682" height="308" alt="image" src="https://github.com/user-attachments/assets/058a4543-775f-4008-9697-fd03c16eabe8" />

We get a confirmed account, admin@mediahub.thm, but the password isnt fully there for us. To find it I build a python script instead of testing each year manually. At first I ran into rate limiting issues when trying to use burpsuite intruder - after around 5 attempts it would block me, but the blocking was based on the phpsessionid - all I had to do is ensure that in the script, each attempt generated a new id to bypass the limiting issue.

<img width="795" height="729" alt="image" src="https://github.com/user-attachments/assets/a5297aa2-56e4-4ad4-8402-451476a5a677" />

After a couple tweaks to the script it ended up running succesfully, and we have credentials 

<img width="1218" height="428" alt="image" src="https://github.com/user-attachments/assets/afe192a9-e2f8-4dc5-85e4-ccbecc18771b" />

After authenticating we are met with a OTP prompt, so I open burp to see whats going on under the hood, we also see that it is posting to verify_otp.php

<img width="686" height="353" alt="image" src="https://github.com/user-attachments/assets/4b17c4ed-b7e5-4ca6-a2c1-21fe68ec5eb1" />

After a couple of failed methods, attempting injections, considering brute forcing, I simply tried to play with the logic


<img width="1285" height="401" alt="image" src="https://github.com/user-attachments/assets/824f49d7-9857-41cb-a066-d38d9db8aa10" />

We see that its boolean logic, true or false, so what if we made it true? my first couple tries didnt work but when switching the form-data to the corresponding json it worked

<img width="1125" height="128" alt="image" src="https://github.com/user-attachments/assets/a395475f-d3c6-48a8-a47b-28753835cf60" />

I get sent to dashboard.php, which has a file upload feature, and a url fetching feature - immediately I think of SSRF or some kind of code execution via the URL import feature

<img width="1372" height="560" alt="image" src="https://github.com/user-attachments/assets/53e52590-080c-400c-ae81-fe927e0c359e" />

After attempting to call my own server, 127.0.0.1, and other attemnpts to pull the contents of the file directly I came across a syntax that bypassed the guardrails and got the flag

<img width="1267" height="273" alt="image" src="https://github.com/user-attachments/assets/570c6ae0-b04c-459a-b308-601761310609" />















