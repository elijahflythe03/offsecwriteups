I start with an nmap scan which identified p80 and p22

<img width="865" height="536" alt="image" src="https://github.com/user-attachments/assets/f5b18a7b-ed26-4ad1-b8ce-06e4ebfa31dd" />

upon inspecting the web page I see its an auth portal

<img width="731" height="460" alt="image" src="https://github.com/user-attachments/assets/96e1fc4f-0fa8-4e5d-8c2f-a220706f7822" />

After inspecting the source code I identify a couple paths, /auth.php and /dashboard.php, along with potential users adam, deliver11, and nora. I also find a check.js file that includes the authentication logic.

<img width="761" height="444" alt="image" src="https://github.com/user-attachments/assets/3eec45bd-df40-4167-b9a0-6dcf030eecf5" />


I next move to further enumerating the paths this website has with ffuf. I run a dir fuzzing scan and uncover more paths

<img width="1063" height="422" alt="image" src="https://github.com/user-attachments/assets/7d455a68-6cd1-4c38-893a-a4b4028aac54" />

Next I test the auth mechanism by sending a standard test input to see how it behaves 

<img width="898" height="301" alt="image" src="https://github.com/user-attachments/assets/eef10b60-88dc-4ca7-94fc-5ff06d96ff08" />

Noted, but when I use one of the confirmed users we get this

<img width="958" height="290" alt="image" src="https://github.com/user-attachments/assets/2af5debc-5b46-40ab-b5f7-8e33fc1859fb" />
<img width="963" height="295" alt="image" src="https://github.com/user-attachments/assets/98f6f0c6-95cd-4624-ab38-f6f017f558f6" />
<img width="951" height="313" alt="image" src="https://github.com/user-attachments/assets/eaef9bda-adf8-4789-a2d2-be4e6d07f9d6" />

It gives us some hints, guessing is kinda trivial so I run hydra in the background to test for credentials against the valid users while I continue recon

hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 10.130.143.125 http-post-form "/auth.php:username=^USER^&password=^PASS^:F=auth_failed" -V

Nikto tells me that /admin.php is potentially vulnerable to an auth bypass, and upon heading there I am prompted with an auth portal







