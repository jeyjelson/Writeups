# Hack The Box - Beep | Write-up

> **Platform:** Hack The Box &nbsp;•&nbsp; **Category:** Linux &nbsp;•&nbsp; **Difficulty:** Easy
>
> **Target:** `10.129.52.183` &nbsp;•&nbsp; **Time taken:** 60 mins
>
> **Author:** Jithin Jelson

---

## Introduction

Beep is a Linux machine and it is rated easy on Hack The Box, we will be trying to exploit it today.

![Hack The Box Beep machine page](images/01-htb-beep-machine.png)

*Figure 1 - The Beep machine on Hack The Box, an easy-rated Linux target.*

---

## Assessment Overview

```mermaid
flowchart LR
    A[Nmap scan<br/>many open ports]:::entry --> B[HTTPS web interface<br/>port 443]:::entry

    B --> C[TLS unsupported<br/>lower tls.min in<br/>about:config]:::intel
    C --> D[Elastix login<br/>+ page source]:::intel

    B --> E[ffuf directory<br/>enumeration]:::intel
    E --> F[admin panel<br/>FreePBX 2.8.1.4]:::intel

    F --> G[searchsploit elastix<br/>graph.php LFI]:::payload
    G --> H[LFI reads<br/>amportal.conf]:::payload
    H --> I[Recovered<br/>admin password]:::payload

    I --> J[SSH with legacy<br/>kex + host key]:::user
    J --> K[Root shell<br/>password reuse]:::user
    K --> L[user.txt +<br/>root.txt]:::user

    classDef entry fill:#1d4ed8,stroke:#1e3a8a,color:#ffffff;
    classDef intel fill:#7c3aed,stroke:#5b21b6,color:#ffffff;
    classDef payload fill:#be123c,stroke:#881337,color:#ffffff;
    classDef user fill:#15803d,stroke:#14532d,color:#ffffff;
    linkStyle default stroke-width:2px
```

---

## What I Learned

- How to read an Nmap scan of a busy host and pick out the web service worth attacking first.
- Working around a browser that refuses an old server by lowering the minimum TLS version in about:config.
- Enumerating web directories with ffuf to uncover an admin panel that leaks the exact software version.
- Using searchsploit to match a known version (Elastix / FreePBX) to a public exploit.
- Abusing a graph.php Local File Inclusion to read amportal.conf and recover plaintext credentials.
- Why password reuse is so dangerous, because the same admin password gave straight root over SSH.

---

## Nmap Scan

We can start off by carrying out an Nmap scan of the target IP address.

![Nmap scan showing many open ports](images/02-nmap-scan.png)

*Figure 2 - Nmap reveals many open services including SSH, SMTP, HTTP, HTTPS, IMAP, POP3, MySQL and port 10000.*

It seems we have a web interface so lets check it out.

---

## Fixing the TLS Version

However when we open the web page it seems that our TLS version isn't supported.

![Firefox Secure Connection Failed error](images/03-tls-connection-failed.png)

*Figure 3 - The browser refuses the connection with SSL_ERROR_UNSUPPORTED_VERSION because the server only offers an old TLS version.*

To fix this we will go to about:config and then change our security.tls.version.min from 3 to 1.

![about:config with security.tls.version.min set to 1](images/04-aboutconfig-tls-min.png)

*Figure 4 - Lowering security.tls.version.min in about:config so Firefox will talk to the old server.*

Now it seems we have access onto our website.

![Elastix login page](images/05-elastix-login.png)

*Figure 5 - The Elastix login page loads once the TLS minimum is lowered.*

---

## Reviewing the Source Code

We can now pay attention to the source code to see if we have anything useful for us.

![View-source of the Elastix login page](images/06-login-source-code.png)

*Figure 6 - Viewing the page source of the Elastix login.*

---

## Directory Enumeration with ffuf

With nothing much useful there we can enumerate our endpoint using ffuf.

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u https://10.129.52.183/FUZZ -o output.txt
```

![ffuf directory enumeration results](images/07-ffuf-enumeration.png)

*Figure 7 - ffuf finds directories including admin, panel, configs and more.*

We can see that we have all the directories to explore.

We checked out the admin panel but it seems it was asking us for credentials and we tried defaults but it never worked, however when we cancelled out it seems we got onto the following page.

![FreePBX 2.8.1.4 Unauthorized page](images/08-freepbx-version.png)

*Figure 8 - Cancelling the login reveals a FreePBX 2.8.1.4 "Unauthorized" page, disclosing the exact version.*

Which revealed the FreePBX version to us, FreePBX 2.8.1.4. With this information lets look up on searchsploit if there is any exploits for our Elastix.

![searchsploit elastix results](images/09-searchsploit-elastix.png)

*Figure 9 - searchsploit lists several Elastix exploits, including the graph.php LFI.*

It seems we have a couple but the interesting one is the graph.php LFI exploit.

---

## Exploiting the graph.php LFI

When we look it up we can see the vulnerable directory, so lets try and access this to see if we get LFI.

![Exploit-DB 37637 graph.php LFI source](images/10-exploitdb-37637-lfi.png)

*Figure 10 - Exploit-DB 37637 shows the vulnerable /vtigercrm/graph.php path used to read amportal.conf.*

And it seems like we were successful.

![LFI output of amportal.conf rendered in browser](images/11-lfi-amportal-rendered.png)

*Figure 11 - The LFI reads amportal.conf, exposing the FreePBX configuration and passwords.*

We can open up the source code of this to find it in a better order as the browser renders it as HTML.

![View-source of the amportal.conf LFI output](images/12-lfi-amportal-source.png)

*Figure 12 - Viewing the source lays out amportal.conf line by line, revealing AMPDBPASS and AMPMGRPASS.*

---

## SSH Access as Root

We can see a pass for admin so lets try and use it in our SSH, however it seems our system is using a legacy key.

![Failed SSH attempts due to legacy key exchange](images/13-ssh-legacy-key-fail.png)

*Figure 13 - The modern SSH client refuses the target's legacy key exchange algorithms.*

So we can try the following.

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa root@10.129.52.183
```

We will enter the repeated password `jEhdIekWmdjE`.

![Successful SSH login as root on Beep](images/14-ssh-root-success.png)

*Figure 14 - Adding the legacy kex and host key algorithms lets us log in, and the reused password gives a root shell.*

And just like that we got root access, now we can find our flags.

---

## Capturing the Flags

![cat root.txt showing the root flag](images/15-root-flag.png)

*Figure 15 - Reading root.txt from the root home directory.*

![user.txt and root.txt collected](images/16-user-and-root-flags.png)

*Figure 16 - Collecting user.txt from /home/fanis alongside the root flag.*

> Submit the user flag.

**Answer:** `d4572c8bd2a57e9033bcfd936527ae3d`

> Submit the root flag.

**Answer:** `1158a8e990560c2b3d008dcc23a4714c`
