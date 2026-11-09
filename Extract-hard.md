I kick it off with an nmap scan, standard port 80 and 22 open

<img width="891" height="222" alt="image" src="https://github.com/user-attachments/assets/103088ea-14e5-41a8-88e1-a7eab02914c2" />

When viewing the website, its a document preview library essentially. While I check out the functionality I run ffuf in the background to identify paths

<img width="1562" height="823" alt="image" src="https://github.com/user-attachments/assets/3f65ce8a-86da-4968-ba0f-ea822d208f9a" />

The functionality is pretty straightfroward, you click on a document, it displays in the little window. My ffuf scan has ran and we have a few paths that could be attack surfaces

<img width="874" height="267" alt="image" src="https://github.com/user-attachments/assets/5db4b64a-04c8-413e-a6a0-7cd787786547" />

Management is auth gated, preview requires a url parameter, so we head to burpsuite. When looking at the request from the preveiw document functionality I see a parameter that looks very injectable.

<img width="714" height="214" alt="image" src="https://github.com/user-attachments/assets/3ae9c9af-88b6-402d-875a-d8bd49bebdf3" />

I send this to repeater to see if we can make any altered requests through the parameter. Firstly I confirm the vuln by connecting back to my own machine, and its succesful

<img width="528" height="138" alt="image" src="https://github.com/user-attachments/assets/192e0a52-3b2f-4aab-a659-905118a3a83b" />

Nice. Now I call the loopback address to see if I can reach internally, and I can. So I think.. can I view the Management path from here? and yes I can. by Pointing to http://127.0.0.1/management I bypass the auth gate and see whats inside

<img width="1534" height="334" alt="image" src="https://github.com/user-attachments/assets/25645118-c3a3-4d69-9843-442024912367" />

Now I am faced with an internal auth portal, which displays in the "preview" window

<img width="933" height="689" alt="image" src="https://github.com/user-attachments/assets/e08629dd-cf4a-4b37-af6c-1706b8d53c58" />

When looking at the auth logic, the portal kind of seems like a distraction. It POSTs back to itself, so I try continue to enumerate /management through its loopback address. No internal path scan is fruitful, but upon running an internal port scan I find port 10000 to be open.

<img width="907" height="425" alt="image" src="https://github.com/user-attachments/assets/ba7d7caa-22af-4f73-a8cd-81f5c8efcac6" />

I view port 10,000 through the SSRF, and I get my next lead, TryBookMe API, AKA /customapi

<img width="761" height="466" alt="image" src="https://github.com/user-attachments/assets/ca6c5b6b-c4ba-458e-acb5-3d01382f9cab" />

After some testing to figure out what we can even do, or how I can interact with this endpoint, I FUZZ a test parameter with ffuf and discover 'dnslocalhost', this is another lead, but I still need to find out how to porperly interact with it

<img width="1278" height="451" alt="image" src="https://github.com/user-attachments/assets/7cabe975-c0cd-4a32-8c32-bbd2ef48e9fb" />

I send a 'test' in the customapi?dnslocalhost= parameter and I get some sort of interaction

<img width="1288" height="295" alt="image" src="https://github.com/user-attachments/assets/4ace4268-d14d-4a14-a946-f2f663935a19" />

It was blocked, but at least its something.














