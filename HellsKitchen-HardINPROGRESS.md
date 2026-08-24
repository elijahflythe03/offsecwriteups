DX2: Hell's Kitchen

I start with a verbose, full port, service enumeration scan 

<img width="678" height="243" alt="image" src="https://github.com/user-attachments/assets/5cb3522b-2b7d-4712-870b-de70bde14d9f" />

while waiting for service enumeration I head over to port 80 to find a booking site. you can book a room, view the guest book, and view the about page

<img width="667" height="480" alt="image" src="https://github.com/user-attachments/assets/885667ae-63f8-459d-8ec1-5c5d08865b58" />

The booking functionality is blocked, but we will figure that out.

<img width="652" height="335" alt="image" src="https://github.com/user-attachments/assets/99429fa8-ab8b-4d3e-ba99-87179eee8d3b" />

In the guest book we see a table, potentially hinting at a sql db

<img width="826" height="466" alt="image" src="https://github.com/user-attachments/assets/1e9ac4b8-361c-45e5-a247-a20614e42647" />

My service scan finishes and we see that on port 4346 there is a web app, a classic auth portal.

<img width="752" height="462" alt="image" src="https://github.com/user-attachments/assets/bba2309c-7117-4b90-995c-49653ff109d9" />

I ran dir and vhost scans on the first web app, nothing fruitful, so I run it against the auth portal and get /mail and /ws. Im sure I will find more in manual enumeration but this is an alright start

<img width="1092" height="445" alt="image" src="https://github.com/user-attachments/assets/6f60c02f-7d55-4105-8eeb-3fa514896556" />

I start by inspecting the buttons on the booking page, I want to see how they work, where they send data or lead to, etc.

When checking the js I find a new path /new-booking

<img width="727" height="252" alt="image" src="https://github.com/user-attachments/assets/661c10b5-0c54-4ed0-a432-4fc0002bf98b" />

Upon navigating to this page, its pretty much blank, but what's to the eye isn't always all there is

<img width="908" height="509" alt="image" src="https://github.com/user-attachments/assets/458bf4b6-1731-4e3b-9d64-d925dcace549" />

When inspecting the page we find a hidden POST form that handles booking, I want to interact with it so to make it render I change the display from none to blocked. 

I also see that we now have a cookie set. 

<img width="907" height="322" alt="image" src="https://github.com/user-attachments/assets/db01d644-017a-4e36-ae73-f60fafc1dc45" />

Keeping note of that

<img width="1295" height="673" alt="image" src="https://github.com/user-attachments/assets/4fd16188-d24b-4dc5-9502-83fa66db7738" />

After manipulating the rendering of the POST form, we are then presented with a nights text box, I want to see if its vulnerable to any kind of injection. When submitting an expected response, I get an internal server error 

<img width="881" height="367" alt="image" src="https://github.com/user-attachments/assets/949d0b31-b59f-4a94-bc9c-169930ce6f8f" />

The exact same thing happens when I submit a dummy xss payload after switching the expected client side input from number to text. Ive seem to hit a wall with this one and I don't want to waste time on rabbit holes, so I head over to the auth portal to do manual enumeration.

When navigating to the auth portal on 4346 I see that I am still given a booking_key cookie.. thats interesting

<img width="448" height="249" alt="image" src="https://github.com/user-attachments/assets/3492f627-b577-4f8a-be22-2a92f6f74b5d" />

I first want to decide the level of entropy and see if there are any differences, new tokens asigned with new visits, etc

<img width="331" height="57" alt="image" src="https://github.com/user-attachments/assets/ff5279fc-24e8-4d7c-93ff-b971fb273e23" />

I see that the first chunk remains the same while the latter half values change, we could potentially fuzz this for IDOR, noted

Before I take those steps I want to test the auth



















