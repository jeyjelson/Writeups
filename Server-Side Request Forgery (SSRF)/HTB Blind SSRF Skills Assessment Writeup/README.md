# Hack The Box Academy - Blind SSRF Enumeration | Write-up

> **Platform:** Hack The Box Academy &nbsp;•&nbsp; **Category:** Server-Side Request Forgery (SSRF) &nbsp;•&nbsp; **Difficulty:** Easy &nbsp;•&nbsp; **Time taken:** 15 minutes &nbsp;•&nbsp; **Target IP:** 10.129.129.53
>
> **Author:** Jithin Jelson

---

## Introduction

In this task we are told to identify what other ports are open other than port 80. We have been told that the vulnerability here is an SSRF vulnerability and it is a blind SSRF, so we do not get much information back from our attack.

---

## Assessment Overview

```mermaid
flowchart LR
    A["Web App<br/>10.129.129.53"] --> B["Identify SSRF<br/>dateserver param"]

    B --> C["Confirm Blind<br/>Point at own homepage<br/>No content returned"]

    B --> D["Baseline Open Port<br/>Port 80<br/>Data is unavailable"]
    B --> E["Baseline Closed Port<br/>Port 81<br/>Something went wrong"]

    D --> F["ffuf Port Scan<br/>127.0.0.1:FUZZ<br/>via dateserver"]
    E --> F

    F --> G["Filter:<br/>-fr 'Something went wrong!'"]
    G --> H["Open Port<br/>Discovered"]

    classDef entry fill:#1d4ed8,stroke:#1e3a8a,color:#ffffff;
    classDef recon fill:#0f766e,stroke:#134e4a,color:#ffffff;
    classDef finding fill:#7c3aed,stroke:#5b21b6,color:#ffffff;
    classDef tool fill:#b45309,stroke:#78350f,color:#ffffff;
    classDef result fill:#be123c,stroke:#881337,color:#ffffff;

    class A entry;
    class B,C recon;
    class D,E finding;
    class F,G tool;
    class H result;

    linkStyle default stroke-width:2px
```

---

## What I Learned

- How to confirm a blind SSRF by pointing the vulnerable parameter at the application's own homepage and observing that no content is returned.
- Distinguishing open from closed ports in a blind SSRF context by comparing the two different error responses the application gives.
- Using ffuf with a port wordlist and the `-fr` flag to filter out closed-port responses and surface only the open ones.
- That the SSRF payload must target the vulnerable parameter directly (`dateserver=http://127.0.0.1:FUZZ`) rather than fuzzing the server's external IP, because only the server itself can reach its own loopback interface.
- Why blind SSRF is still a useful enumeration primitive even without direct data exfiltration, since internal port scanning can reveal hidden services for follow-on exploitation.

---

## Identifying the Vulnerability

First I visited the web application to understand its functionality.

![Web app homepage](images/01-web-app-homepage.png)
*Figure 1 - The web application homepage with the date availability checker*

The web page is the same as in previous exercises where I identified the vulnerability in the availability section, so most likely it is here again. I confirmed this using Burp Suite and sending a request to Netcat.

![Burp Suite confirming SSRF in dateserver parameter](images/02-burp-ssrf-confirmation.png)
*Figure 2 - Burp Suite confirming the SSRF vulnerability via the dateserver parameter*

---

## Confirming Blind SSRF

I confirmed the vulnerability is in the same area. I then checked whether it was blind by pointing it at the application's own homepage.

![Blind SSRF confirmed - no content returned](images/03-blind-ssrf-confirmed.png)
*Figure 3 - Pointing the SSRF at its own homepage returns "Data is unavailable" with no page content, confirming blind SSRF*

When pointing it at its own homepage, instead of displaying the index page it says "Data is unavailable". Since it is not displaying anything to us, we can confirm it is blind SSRF.

---

## Establishing Port Response Baselines

I then tried to probe the server to see what ports are available. Since the web server is running HTTP, I knew port 80 would be available, so I tried that first.

![Port 80 open - Data is unavailable response](images/04-port-80-open-response.png)
*Figure 4 - Probing port 80 returns "Data is unavailable", establishing the open-port response signature*

It gives the same message as before, which means this message most likely indicates something is available. I then checked for a different response by probing a commonly closed port, in this case port 81.

![Port 81 closed - Something went wrong response](images/05-port-81-closed-response.png)
*Figure 5 - Probing port 81 returns "Something went wrong!", establishing the closed-port response signature*

Now I can see the error messages differ. If a port is open it gives "Data is unavailable" and if it is closed it gives "Something went wrong!". Using this information I can answer the question by putting a few filters into ffuf.

---

## Port Scanning with ffuf

> Exploit the SSRF to identify open ports on the system. Which port is open in addition to port 80?

First I created a wordlist of all ports and input it into ffuf. Note: the first command used here was wrong as I had not specified where the vulnerability was, and was just pointing ffuf directly at the website's external IP rather than routing through the vulnerable parameter.

![Initial wrong ffuf command](images/06-ffuf-wrong-command.png)
*Figure 6 - Initial ffuf command incorrectly scanning the external IP rather than exploiting the SSRF parameter*

The corrected command routes the probe through the vulnerable `dateserver` parameter so the server scans its own loopback interface, reaching ports that are not externally accessible:

```bash
ffuf -w ports.txt \
     -u http://10.129.129.53/index.php \
     -X POST \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "dateserver=http://127.0.0.1:FUZZ&date=2024-01-01" \
     -fr "Something went wrong!"
```

![Correct ffuf command results showing open port](images/07-ffuf-correct-command-results.png)
*Figure 7 - Corrected ffuf command exploiting the dateserver parameter to scan internal ports, revealing the open port*

**Answer:** `(see Figure 7)`
