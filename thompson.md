# TryHackMe - Tomcat Writeup

## Initial Recon

First thing I did was add the ip to my hosts file.

<img width="594" height="97" alt="image" src="https://github.com/user-attachments/assets/7b6a655c-165e-444c-bc5a-8b9560bccef3" />


Next I ran a network scan to begin information gathering.

<img width="975" height="740" alt="image" src="https://github.com/user-attachments/assets/aad79647-ce53-44f8-bff7-945f13947c13" />


I see ports 22, 8009, and 8080 open, so lets check out the webpage.

<img width="975" height="702" alt="image" src="https://github.com/user-attachments/assets/67439c3d-70c2-4d35-968e-7c5382fc98c3" />


Its pretty simple, I'm keeping note of Apache Tomcat version 8.5.5 just incase there is any vuln associated with the version.

## Manual Enumeration

Next I manually pry through the page. When I click on the manager app button it actually requires auth.

<img width="975" height="714" alt="image" src="https://github.com/user-attachments/assets/51f91574-b3b8-44f6-895f-6f237ff595c4" />



Luckily for us, when the unauth error pops up it actually leaks the credentials.

<img width="975" height="714" alt="image" src="https://github.com/user-attachments/assets/9fb1bda4-0038-47d0-9b86-531abf840a9f" />


We then enter the manager portal, which looks like this.

<img width="975" height="716" alt="image" src="https://github.com/user-attachments/assets/ae0f45f7-b4c7-4c25-abe8-94da3db8dfeb" />


## Exploitation

First thing I noticed is a WAR file upload area — this has to be an attack surface. I check searchsploit to see if there are any low hanging exploits associated and no luck.

<img width="488" height="134" alt="image" src="https://github.com/user-attachments/assets/8774df07-e862-48b6-a23d-62a1b9522a23" />


No luck there, so we take the semi-manual route with msfvenom payload generation.

<img width="975" height="94" alt="image" src="https://github.com/user-attachments/assets/2db3d7d1-376a-44a9-b778-a122edbcfa46" />


<img width="975" height="61" alt="image" src="https://github.com/user-attachments/assets/736711ae-082d-4453-9e2a-dc7c04a1b9c6" />


File upload complete, now lets spin up our listener and hope for a callback.

<img width="702" height="228" alt="image" src="https://github.com/user-attachments/assets/f5d59cb6-c9dd-413a-aa64-5c1d77a910ad" />


## \#wearein

Now lets start Linux enumeration!

Firstly lets stabilize this shell.

<img width="695" height="200" alt="image" src="https://github.com/user-attachments/assets/468453fd-041c-49d8-af70-5a31d03850ce" />


## Privilege Escalation

After some enumeration we find the user flag under the user jack.

<img width="839" height="602" alt="image" src="https://github.com/user-attachments/assets/ebd58d09-9b98-48eb-8344-95171c9f0603" />
)

Next, lets dive deeper for the root flag. The `id.sh` and `test.txt` look interesting.

<img width="402" height="158" alt="image" src="https://github.com/user-attachments/assets/b64d1156-80f2-4a88-82d0-459ecfad532e" />


We figure out that it's a cronjob that exports id results to `test.txt`, with root privileges that we can alter.

<img width="975" height="333" alt="image" src="https://github.com/user-attachments/assets/88f927a0-6cd3-4db3-a28e-e497b277020d" />


This is confirmation of our privilege escalation vector.

<img width="761" height="198" alt="image" src="https://github.com/user-attachments/assets/02791fa9-9794-4708-917c-dc0104166e61" />


I ended up altering the script so that it outputs the contents of root to `test.txt`. This way we can bypass gaining total root access to view the flag. Challenge completed!
