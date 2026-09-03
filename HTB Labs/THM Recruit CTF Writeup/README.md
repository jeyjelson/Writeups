# TryHackMe - Recruit | Write-up

> **Difficulty:** Intermediate &nbsp;•&nbsp; **Category:** Web &nbsp;•&nbsp; **Topics:** Enumeration · LFI · SQL Injection
>
> **Author:** Jithin Jelson

---

This was my first intermediate challenge on TryHackMe, however it was fairly straightforward and the methodology for the process was easy compared to other HackTheBox labs. The recommended time was 60 minutes, but it took maybe 30-45 minutes to complete. The exercise focuses on basic enumeration to retrieve the user flag and simple SQL injection to retrieve the admin flag.

---
## What I Learned

- feroxbuster recursively enumerates into the directories it finds with a single command, giving a better sitemap than a single-level enumeration
- the cv parameter only allowed local files, so I had to point it at a local file path to make the LFI work
- LFI could be used to read config.php and reveal the login credentials
- How to do an error-based in-band SQL injection

---

## Attack overview

```mermaid
flowchart LR
    A["Target\nTryHackMe machine"] --> B

    subgraph recon ["Reconnaissance"]
        B["nmap -sV\n3 ports open"]
    end

    B --> C

    subgraph enum ["Enumeration"]
        C["Feroxbuster\nrecursive dir scan"]
        C --> D["/mail directory\ncredential hint found"]
    end

    D --> E

    subgraph lfi ["LFI - user flag"]
        E["cv= parameter\nlocal file inclusion"]
        E --> F["config.php leaked\nusername + password"]
        F --> G["User flag captured"]
    end

    G --> H

    subgraph sqli ["SQLi - admin flag"]
        H["Login page\nvulnerable to SQLi"]
        H --> I["UNION SELECT\n4 columns confirmed"]
        I --> J["information_schema\nusers table dumped"]
        J --> K["Admin flag captured"]
    end

    style recon fill:#f5f0ff,stroke:#8b5cf6,color:#3c1f7a
    style enum fill:#e8f4fd,stroke:#2980b9,color:#1a3a4f
    style lfi fill:#e8fdf4,stroke:#059669,color:#064e3b
    style sqli fill:#fef3e2,stroke:#e67e22,color:#7d4000
    style G fill:#e8fdf4,stroke:#059669,color:#064e3b
    style K fill:#fde8e8,stroke:#c0392b,color:#7b0000
```

## Initial Reconnaissance

First and foremost, we want to verify if the target is accessible.

![Verifying connectivity to the target](images/01-ping-target.png)
*Figure 1 - Verifying connectivity to the target*

Now that we have our target accessible, we can run an nmap scan with `-sV` to see the open ports and service versions.

![Nmap service-version scan](images/02-nmap-scan.png)
*Figure 2 - Nmap service-version scan*

We can see that we have 3 ports open. Since there is a web page open, we can visit it, and we are presented with a recruiter login page.

![Recruiter login page](images/03-recruiter-login.png)
*Figure 3 - Recruiter login page on port 80*

The recruiter login has an 'access api' page which has some useful information. We can see the parameter required to fetch a file.

![API access page](images/04-api-access.png)
*Figure 4 - API access page exposing the `cv` parameter*

---

## Directory Enumeration

There is nothing else of much use that we have direct access to, so our next step is to enumerate. I usually use ffuf and gobuster, but I recently came across feroxbuster, which reliably enumerates with just a single command. The tool also doesn't stop at an initial enumeration but it further enumerates into the directories it found, giving me a better sitemap.

![Feroxbuster recursive enumeration](images/05-feroxbuster.png)
*Figure 5 - Feroxbuster recursive enumeration*

We have an interesting directory here. We come across a mail directory and, as we suspected, there is content in the mail directory that guides us to the next step of the process.

![Contents of the /mail directory](images/06-mail-directory.png)
*Figure 6 - Contents of the `/mail` directory*

---

## Local File Inclusion - User Flag

We tried to access `config.php` but it wasn't a directory that we can view. So we remembered from earlier when we went to the API Access page, we saw a way we can search for documents using the parameter.

![Attempting to fetch config.php](images/07-lfi-error.png)
*Figure 7 - Attempting to fetch `config.php` via the `cv` parameter*

However, we got an error message saying only local files are allowed, therefore it means we need to put the local destination after the parameter, so we did this.

![Successful LFI revealing credentials](images/08-lfi-success.png)
*Figure 8 - Successful LFI revealing credentials*

Just like that, we came across both the username and the password for a login, and when we login we get our first flag.

![User flag captured](images/09-user-flag.png)
*Figure 9 - User flag captured*

---

## SQL Injection - Admin Flag

If we paid attention to the mail from earlier, we can see that they say the admin credentials were stored in a database, so my first instinct was to use SQL injection to retrieve this. First, I tried it on the recruit login page but it didn't seem to work, so I tried it on the new page where we got our flag.

I decided to test this by breaking it with `'`

And as we suspected, it had given us a SQL error, which means the app is vulnerable to SQL injection.

![MySQL error confirming injection point](images/10-sql-error.png)
*Figure 10 - MySQL error confirming injection point*

So then I used the union payload to check for the number of columns. This is a classic example of Error-based in-band SQL injection.

```sql
' union select 1,2,3,4-- -
```

Using this and increasing the number step by step, we can confirm there are 4 columns.

Next, we fetched the tables from the database using the `from information_schema.tables`.

And we returned a table named `users`. So we decided to view this using the following payload:

```sql
' union select column_name,2,3,4 from information_schema.columns where table_name='users'-- -
```

The password column was found, and this can now be used to get the admin credentials.

```sql
' union select 1,password,3,username from users-- -
```

![Admin credentials extracted](images/11-admin-credentials.png)
*Figure 11 - Admin credentials extracted from the users table*

Now we can use these credentials to retrieve the admin credentials.

![Admin flag captured](images/12-admin-flag.png)
*Figure 12 - Admin flag captured*

---

<sub>Write-up by <b>Jithin Jelson</b></sub>
