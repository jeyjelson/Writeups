# XSS - Cross Site Scripting

> **Category:** Web Application Penetration Testing &nbsp;•&nbsp; **Vulnerability:** Cross-Site Scripting (XSS) &nbsp;•&nbsp; **Author:** Jithin Jelson &nbsp;•&nbsp;**Difficulty:** Easy/Medium

In this assessment we are performing a web application penetration testing task for a company that has hired us, they have just released their security blog, in our web app penetration testing we have reached the part where we must test the web application against cross-site scripting vulnerabilities. The instructions we have been given is to identify a user input field that is vulnerable to an xss vulnerability in the assessment directory, find a working xss payload that executes javascript code on the targets browser and use session hijacking techniques to steal the victims cookies which contains the flag.

- **My IP:** `10.10.15.39`
- **Target IP:** `10.129.73.171`

---

## What I Learned

- That input sanitisation was stripping out the script tag and the ">" character
- That a comment waiting moderation meant I had to attempt a blind xss, as it was being carried out in a panel I don't have access to
- That netcat is not an actual web server, it just opens a raw connection and can't send back a proper response, so the browser never received my script.js
- That PHP's built in server with `php -S 0.0.0.0:8080` acts like a proper web server, so it can serve script.js and stay running to catch multiple requests

---
## Attack Overview
```mermaid
flowchart TD
    Start([Security Blog]):::start --> Test{Test input<br/>fields}:::decision

    Test -->|comment| B1[script tag<br/>stripped]:::blocked
    Test -->|search bar| B2[no callback]:::blocked
    Test -->|**url field**| V[**Vulnerable**<br/>blind XSS]:::good

    V --> CB[Netcat callback<br/>received]:::good
    CB --> Serve{Serve<br/>script.js}:::decision
    Serve -->|netcat| F[fails, not a<br/>web server]:::blocked
    Serve -->|**php -S**| P[serves script]:::good
    F -.switch.-> P
    P --> Flag([Cookie stolen<br/>Flag captured]):::flag

    classDef start fill:#1e293b,stroke:#475569,color:#e2e8f0
    classDef decision fill:#1e3a5f,stroke:#3b82f6,color:#dbeafe
    classDef blocked fill:#3f1d2e,stroke:#e11d48,color:#fecdd3
    classDef good fill:#14342b,stroke:#10b981,color:#d1fae5
    classDef flag fill:#14342b,stroke:#10b981,color:#d1fae5,stroke-width:2px
```

## Reconnaissance

Normally the first step would be to carry out an nmap scan but the assessment has already provided us with instructions on where we should find our vulnerability so we can go to that page.

![Assessment page](images/01-assessment-page.png)
*Figure 1 - Navigating to the provided assessment page*

After a quick reconnaissance of our web page we came across a section we could leave a comment behind, this seems to be our best find so far.

![Comment section](images/02-comment-section.png)
*Figure 2 - Comment section identified as a potential injection point*

---

## Testing the Comment Field

After entering our initial payload of `<script>alert(1)</script>` we find out that some input sanitisation is being taken place, we know this as I clicked on save my information and the input fields had changed my text.

![Payload before submission](images/03-payload-before.png)
*Figure 3 - Payload before submission*

![Payload after submission](images/04-payload-after.png)
*Figure 4 - Payload stripped after submission*

It seems after closer inspection the script tag is being stripped out by input sanitisation, so we can test another payload instead using the github repo PayloadAllTheThings.

It seems to be that the only injection point is the comment field or username field as the url field has url field validation assigned to it.

![Field validation](images/05-field-validation.png)
*Figure 5 - The url field has url validation assigned to it*

And the email field also has an email field validation assigned to it.

This was our next payload.

![Second payload attempt](images/06-second-payload.png)
*Figure 6 - Second payload attempt using PayloadAllTheThings*

This time when I checked my comment there was nothing to be found.

![Empty comment](images/07-empty-comment.png)
*Figure 7 - Comment field empty after submission*

This was interesting, maybe this means the payload had worked, also it says comment is waiting moderation so maybe I had to attempt a blind xss? As the moderation was being carried out in maybe an admin panel or a panel we don't have access to?

---

## Attempting Blind XSS

So with these questions in mind I attempted a blind xss payload and I started up my netcat listener.

![Netcat listener started](images/08-netcat-listener.png)
*Figure 8 - Starting netcat listener for blind XSS*

Unfortunately this was not a success, so my next step was to see how the input was handling my requests.

I attempted numerous payloads and none of them seemed to be a success so I started thinking, the url field still allows me to enter a malicious payload but it gets reworded later on into url format, maybe we can try and bypass this?

![URL field payload attempt](images/09-url-field-attempt.png)
*Figure 9 - Attempting to bypass url field reformatting*

But with no success, it seems to be that the ">" is being stripped and we can't put that in our final message.

Then when I scrolled down a bit I found a search bar, maybe this was an injection point.

![Search bar found](images/10-search-bar.png)
*Figure 10 - Search bar identified as a possible injection point*

Seems like there is no input sanitisation on the search bar, this one seems to look promising, so I modified my payload slightly.

![Modified payload in search bar](images/11-modified-payload.png)
*Figure 11 - Modified payload tested in the search bar*

The field seems to be in a span tag, lets try and close the span tag and then execute our payload.

At this point I was stuck and I didn't know what to do so I reached out for the hint on hackthebox and it said you can't see me but I can see you, which indicated that we are looking for a blind xss.

And finally we got a response from our netcat in the url field after trying out numerous payloads.

![Netcat response received](images/12-netcat-response.png)
*Figure 12 - Response received on netcat listener from url field*

---

## Session Hijacking

Now its time to inject our vulnerability and obtain the flag, since we have only been asked to get the cookie we will write a simple script in which we can get the cookie details and name it script.js.

![Writing script.js](images/13-script-js.png)
*Figure 13 - Writing script.js to capture cookie details*

I used a netcat listener at this point and after some troubleshooting I found out that it doesn't work, and when I researched it I found out that netcat is not an actual web server, it just opens a raw connection and shows you whatever comes through it. It doesn't know how to send back a proper response, so when the browser requested my script.js file it never actually received anything back, meaning the script never ran and the cookie was never sent anywhere.

Because of this I switched over to using PHP's built in server instead with `php -S 0.0.0.0:8080`, since this actually acts like a proper web server, it can serve my script.js file correctly and stays running to catch multiple requests, which is what I needed to both serve the script and capture the cookie.

![Cookie captured](images/14-cookie-captured.png)
*Figure 14 - Successfully capturing the victims cookie*

Then we successfully got the cookie.

---

<sub>Write-up by Jithin Jelson</sub>
