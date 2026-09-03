# Hack The Box - Server-Side Attacks Skills Assessment | Write-up

> **Platform:** Hack The Box &nbsp;•&nbsp; **Category:** Server-Side Attacks &nbsp;•&nbsp; **Difficulty:** Intermediate
>
> **Target:** `154.57.164.67:32670` &nbsp;•&nbsp; **My IP:** `10.10.15.43`
>
> **Author:** Jithin Jelson

---

## Introduction

In this assessment a food truck company named Flavour Fusion Express has asked us to perform a security assessment of its newly launched website. The site aims to improve user engagement and brand presence, however the company is particularly worried about potential server-side vulnerabilities that could compromise sensitive business data. Our task is to evaluate the backend infrastructure, configuration and server logic for weaknesses that an attacker could exploit.

---

## Assessment Overview

```mermaid
flowchart LR
    A[Target website<br/>Flavour Fusion Express]:::entry --> B[View page source<br/>spot XSLT + SSRF]:::intel
    B --> C[POST requests<br/>seen in Burp]:::intel

    C --> D[SSRF testing]:::payload
    D --> D1[netcat catch fails<br/>outbound firewalled]:::ioc
    D --> D2[Probe localhost index<br/>gets a response]:::user
    D1 --> E[In-band SSRF<br/>confirmed]:::mitre
    D2 --> E

    C --> F[Enumeration]:::intel
    F --> F1[ffuf<br/>no endpoints]:::ioc
    F --> F2[Ports 80 + 3306]:::ioc

    C --> G[SSTI testing]:::payload
    G --> G1[Parameter reflects<br/>input]:::intel
    G1 --> G2[Polyglot breaks<br/>template]:::payload
    G2 --> H[Engine identified<br/>Twig]:::mitre
    H --> I[Twig RCE<br/>filter system]:::payload
    I --> J[Read flag.txt<br/>double URL-encoded]:::user

    classDef entry fill:#1d4ed8,stroke:#1e3a8a,color:#ffffff;
    classDef ioc fill:#0f766e,stroke:#134e4a,color:#ffffff;
    classDef intel fill:#7c3aed,stroke:#5b21b6,color:#ffffff;
    classDef mitre fill:#b45309,stroke:#78350f,color:#ffffff;
    classDef payload fill:#be123c,stroke:#881337,color:#ffffff;
    classDef user fill:#15803d,stroke:#14532d,color:#ffffff;
    linkStyle default stroke-width:2px
```

---

## What I Learned

- How to confirm an in-band SSRF by pointing the request back at the target's own localhost when outbound connections are firewalled off.
- Reading a page's source to spot server-side injection points like XSLT and SSRF before ever touching Burp.
- Using a polyglot payload to break a template and then narrowing down the exact engine with an identification chart.
- Why a Twig SSTI can be escalated to remote code execution through the `filter('system')` sink.
- That a space in a payload can break the request, and double URL-encoding it gets the command through.
- Chaining recon, SSRF, SSTI and RCE together into a single path to reach the flag.

---

## Reaching the Target

This skills assessment has told us where the vulnerability lies and it is within the webpage of the target IP address, so we can navigate there first.

![Target homepage](images/01-target-homepage.png)

*Figure 1 - The Flavour Fusion Express homepage on the target.*

We tried to access the menu options but it seems that none of them will actually load.

![Menu options not loading](images/02-menu-not-loading.png)

*Figure 2 - The menu options on the site fail to load.*

So we decided to open up Burp Suite and still no requests were being made when we clicked on it, so we decided to reboot the target and the VM.

---

## Reading the Source

After I loaded up the VM I decided to view the source code of the page and it seems like there is a potential XSLT injection as well as an SSRF vulnerability. It seems like we can make POST requests.

![Page source showing XSLT injection and SSRF](images/03-page-source-xslt-ssrf.png)

*Figure 3 - The page source hints at a potential XSLT injection and SSRF, and shows POST requests can be made.*

![Burp showing POST requests](images/04-burp-post-requests.png)

*Figure 4 - The POST request captured in Burp Suite.*

Upon closer inspection on Burp we can see that new POST requests were coming in one by one.

---

## Confirming the SSRF

![SSRF testing in Burp](images/05-ssrf-testing-burp.png)

*Figure 5 - Testing the suspected SSRF through Burp.*

This seems like an SSRF vulnerability so let's try and catch it on netcat to confirm it. I tried a couple of times to catch it on my netcat listener at port 8000, but it seems like every time I did it Burp would time out, so we decided to try another test which was to point it towards its own localhost index page to see if we get a response or not.

![netcat listener timing out](images/06-netcat-timeout.png)

*Figure 6 - The outbound netcat catch on port 8000 times out.*

This time it seemed to work, maybe the web app has a firewall preventing outside connections from connecting? Anyways with this new information we can confirm that there is an in-band SSRF vulnerability, we can now exploit this.

![localhost probe returning a response](images/07-localhost-probe-response.png)

*Figure 7 - Pointing the request at the target's own localhost returns a response, confirming an in-band SSRF.*

---

## Enumeration and Testing for SSTI

We tried to enumerate using ffuf but it came empty handed, we couldn't find any more endpoints. Next we could try and enumerate the ports but we didn't find much useful stuff, only port 80 and 3306 were open. So I decided to test for SSTI injection in the parameter of the URL.

![ffuf and port enumeration](images/08-ffuf-port-scan.png)

*Figure 8 - Enumeration with ffuf and the ports, only 80 and 3306 are open.*

It seems when we put a random character, in this instance we put the word fusion, it comes back with a response, so we can inject a polyglot to test for whether it will be broken by SSTI.

![SSTI parameter reflection](images/09-ssti-parameter-reflection.png)

*Figure 9 - The parameter reflects our input back in the response.*

---

## Identifying the Template Engine

It seems when we put our polyglot the web app is acting up, so we can methodically try and identify the template engine using the chart we used earlier in identifying a template engine.

![Polyglot breaking the app](images/10-polyglot-breaks-app.png)

*Figure 10 - The polyglot payload breaks the template and the web app acts up.*

We started to have some success.

![Working through the identification chart](images/11-identifying-engine.png)

*Figure 11 - Methodically working through the template engine identification chart.*

Finally we have identified our template engine is Twig.

![Twig identified](images/12-twig-identified.png)

*Figure 12 - The template engine is confirmed to be Twig.*

---

## Remote Code Execution and the Flag

So now that we have our template engine we can try and execute remote code execution.

![RCE executed](images/13-rce-executed.png)

*Figure 13 - Achieving remote code execution through the Twig SSTI.*

We successfully executed RCE, now we can find the flag using ls and cat.

![Bad URL on the space character](images/14-flag-space-issue.png)

*Figure 14 - Reading the flag with ls and cat, but the space character causes a bad URL.*

However just when I thought I was finished it kept giving me a bad URL for the space character, however when I URL encoded it twice it seemed to work.

![Double URL-encoding the payload](images/15-double-url-encode.png)

*Figure 15 - Double URL-encoding gets the command through.*

The final command was:

```
{{['cat%2b../../../flag.txt']|filter('system')}}
```

![Flag retrieved](images/16-flag-retrieved.png)

*Figure 16 - The final payload runs and the flag is retrieved.*

> Retrieve the flag from the target.

**Answer:** `see Figure 16`
