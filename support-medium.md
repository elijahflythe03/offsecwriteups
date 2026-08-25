support.thm

i started with a verbose network scan and quickly identified a web app on port 80 and ssh on port 22.

<img width="822" height="608" alt="image" src="https://github.com/user-attachments/assets/e8f395ac-11e7-4aec-b448-1607f10633c3" />

When viewing the web app its an employee auth portal for a support operations panel. We see a "contatc IT Operations @ help@support.thm" and note that down as a potential account. 

<img width="1720" height="576" alt="image" src="https://github.com/user-attachments/assets/020afa6c-18e3-40d8-906a-b023fee1a9db" />

After additional reconnaisance I uncover a couple hidden routes using nikto, one being config.php and info.php. 

<img width="1091" height="171" alt="image" src="https://github.com/user-attachments/assets/993f6b6c-3bba-4b5f-bceb-b32551fdf61a" />

When viewing info.php it shows an enormous amount of system specifications and information

<img width="1054" height="1056" alt="image" src="https://github.com/user-attachments/assets/9bc28f3e-139f-4145-994c-10d418bb98de" />

we cant really do anything with this information unauthenticated, so we keep moving. Now my target is auth bypass or surfacing credentials.

I try a number of sql and xss payloads but nothing triggers any errors or abnormalities, I then use hydra along with the valid account we know of (help@support.thm) to brute force credentials with the rockyou.txt wordlist and uncover the password for this account!

<img width="906" height="159" alt="image" src="https://github.com/user-attachments/assets/665dcdd4-8588-4eb0-a457-b9ea6181ccf9" />

when entering its simply a ticket management system.. but its a blank page. 

<img width="1769" height="448" alt="image" src="https://github.com/user-attachments/assets/6da54dff-005b-45a3-b4d2-adb39563c9ee" />

After inspecting the HTML line by line I don't see anything interesting, the only functionality on this entire page is the ability to select the theme of this page.

<img width="301" height="241" alt="image" src="https://github.com/user-attachments/assets/0a5a7b30-c2f1-4545-a7fa-43116b599e54" />

after toying with it I notice something in the url, the ?skin= parameter and its potential to test for injection 

<img width="334" height="44" alt="image" src="https://github.com/user-attachments/assets/be22c37e-a202-4c26-bff6-5f362ccab81d" />

After hundreds of manual and automated parameter injection nothing comes back fruitful. I shift my attention to the only other possible attack surface, the isITUser md5 hash. Upon decrpyting we see that it is "false", we change that and resubmit that through a request to /dashboard.php and receive access to the admin panel


<img width="930" height="312" alt="image" src="https://github.com/user-attachments/assets/961efd20-1e2b-4552-b122-f3d7fb8b1a7b" />

seemingly this admin panel gives us access to api.php, which was giving us access denied errors previously. 

<img width="1388" height="305" alt="image" src="https://github.com/user-attachments/assets/880fdc26-2dec-405a-8115-cf35f4f68612" />

api.php is a user guide for the internal api, we notice we can "query your own profile" with /user/3, so that makes me think are there users 1, 2, 4, 5, etc?

<img width="1344" height="468" alt="image" src="https://github.com/user-attachments/assets/fac23fea-0045-44f3-b21a-8114f5f3cfd8" />

Firstly I test my own profile before I begin testing for IDOR

<img width="1045" height="302" alt="image" src="https://github.com/user-attachments/assets/10251017-0bb0-4bdf-a9b5-aa031f0405c9" />

Confirmation that the api is responding as expected, now im going to try to view other users sequentially. I try 1, as it would make sense to and it returns us admin email information

<img width="1064" height="281" alt="image" src="https://github.com/user-attachments/assets/ac9f18e0-52d0-474f-aa0f-4d0a68797632" />

I am shown a different user account, the admin one. Now, my main target is a password for this admin account since thats where the first flag is located. I find my way back to my nikto scan, which told me that there are potentially credentials in the /config dir, so after a couple attempts of lfi on the skin parameter we figure out that ../config shows the content.

<img width="1212" height="480" alt="image" src="https://github.com/user-attachments/assets/a3dd53b5-e625-4263-9de5-83de74a3ac89" />

Using these credentials I log into the admin account which gives us the first flag

<img width="1342" height="498" alt="image" src="https://github.com/user-attachments/assets/f2f33e25-6647-41bc-9d00-c6f8f28c6525" />

Next I attempt to find user.txt from this current session, in the admin portal I see a new 'date' and 'time' button, and upon testing its functionality we see another parameter 'sys' come into play 

<img width="804" height="206" alt="image" src="https://github.com/user-attachments/assets/40a16ae6-0814-47db-9670-2cf567c94229" />

This came only after logging in as admin, so from these privs I will try to exfiltrate the contents of /home/ubuntu/user.txt. After researching the proper injection syntax of the sys parameter I land on sys=date; cat /home/ubuntu/user.txt, which returns the flag!

<img width="1210" height="539" alt="image" src="https://github.com/user-attachments/assets/5c26654b-cb49-4998-a44e-14df756987ac" />






 













