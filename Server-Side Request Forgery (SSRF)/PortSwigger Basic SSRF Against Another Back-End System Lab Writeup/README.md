# PortSwigger - Basic SSRF Against Another Back-End System | Write-up

> **Platform:** PortSwigger &nbsp;•&nbsp; **Category:** Server-Side Request Forgery (SSRF) &nbsp;•&nbsp; **Difficulty:** Apprentice (Easy)
>
> **Target:** `192.168.0.0/24:8080` (internal)
>
> **Author:** Jithin Jelson

---

## Introduction

In this lab we have been told that a stock check feature is vulnerable to SSRF. The stock check feature checks data from an internal server. We have been instructed to scan the internal 192.168.0.x range for an admin interface on port 8080 and delete the user carlos.

---

## Assessment Overview

```mermaid
flowchart LR
    A[Stock check<br/>feature]:::entry --> B[Intercept in Burp<br/>stockApi param]:::payload

    B --> C[URL decode<br/>internal request<br/>192.168.0.1:8080]:::intel

    C --> D[Send to Intruder]:::payload
    C --> H[Send to Repeater]:::payload

    D --> E[Sweep last octet<br/>1-255]:::payload
    E --> F1[Most hosts:<br/>500 error]:::intel
    E --> F2[.139:<br/>404 - host alive]:::intel

    F2 --> H
    H --> I[Point at<br/>192.168.0.139:8080/admin]:::user
    I --> J1[Read admin panel<br/>find delete URL]:::user
    J1 --> J2[Delete carlos<br/>lab solved]:::user

    classDef entry fill:#1d4ed8,stroke:#1e3a8a,color:#ffffff;
    classDef intel fill:#7c3aed,stroke:#5b21b6,color:#ffffff;
    classDef payload fill:#be123c,stroke:#881337,color:#ffffff;
    classDef user fill:#15803d,stroke:#14532d,color:#ffffff;
    linkStyle default stroke-width:2px
```

---

## What I Learned

- How a stock check feature that fetches data from an internal server can be abused to reach systems the client should never touch.
- Using Burp's decoder to unwrap a URL encoded parameter and see the real request being made behind the scenes.
- Running Burp Intruder to sweep an internal 192.168.0.x range and pick out the one host that answers differently.
- Reading response codes carefully, so a 404 or an internal server error tells me whether a host is present and whether the client is simply unauthorised.
- Pivoting the discovered internal request into Repeater to hit the admin panel directly through the SSRF.

---

## Finding the Stock Check Request

So first we can visit the page where this stock check functionality is available and fire up Burp.

![Stock check product page](images/01-stock-check-page.png)

*Figure 1 - The product page with the stock check feature that talks to an internal server.*

Now we can turn on intercept and check for the request being sent to the internal system.

![Intercepted stock check request in Burp](images/02-intercepted-request.png)

*Figure 2 - The intercepted request, with the internal URL held in a URL encoded stockApi parameter.*

We can see that it is URL encoded so we can decode it to see what the request is clearly.

![Decoded stockApi parameter](images/03-decoded-request.png)

*Figure 3 - Decoding the parameter reveals the request going to an internal host on port 8080.*

---

## Scanning the Internal Range

We can see it clearly now. The question tells us it can be anywhere in the IP range 192.168.0.x so we can send this to Intruder.

In Intruder we can send the attack onto the number 1 and create a list from 1-255 that scans the entire network to see which one of it gives us a response.

![Burp Intruder scanning 1-255](images/04-intruder-scan.png)

*Figure 4 - Intruder sweeping the last octet from 1 to 255 across the internal range.*

We seemed to have gotten an internal server error for every IP range except 139 so we can try use SSRF to get into the admin panel this way, as 404 just means we are unauthorised from the client side.

![Differing response at .139](images/05-internal-response.png)

*Figure 5 - Host .139 stands out from the rest of the range with a different response.*

---

## Reaching the Admin Panel and Solving

So we can send this to Repeater.

![Admin panel reached in Repeater](images/06-repeater-admin-panel.png)

*Figure 6 - Pointing the stockApi request at 192.168.0.139:8080/admin reaches the internal admin interface.*

Now we can see the URL required to delete the user so we can put this into our stock API and solve the lab.

![Lab solved after deleting carlos](images/07-lab-solved.png)

*Figure 7 - Sending the delete request for carlos through the SSRF solves the lab.*
