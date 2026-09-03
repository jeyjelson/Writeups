# Hack The Box - PHP Filters: Source Code Disclosure | Write-up

> **Platform:** Hack The Box &nbsp;•&nbsp; **Category:** Web / File Inclusion &nbsp;•&nbsp; **Difficulty:** Easy
>
> **Target:** `154.57.164.75:30373` &nbsp;•&nbsp; **Time taken:** 15 mins
>
> **Author:** Jithin Jelson

---

## Introduction

In this lab we have been instructed to enumerate our web application to find a PHP file, which we must use our PHP filters to convert into base64 and then reveal the database username and password. While PHP filters come in handy for developers, attackers can exploit this feature to gain access and read sensitive information. We must convert to base64 in order to read the application so the PHP does not execute on the page.

---

## Assessment Overview

```mermaid
flowchart LR
    A["Browse target<br/>Inlane Freight :30373"]:::entry --> B["Enumerate with ffuf<br/>FUZZ.php wordlist"]:::payload
    B --> C["Hidden page found<br/>configure.php"]:::payload
    C --> D["Direct include<br/>language=configure<br/>blank, PHP executes it"]:::payload
    C --> E["php://filter wrapper<br/>convert.base64-encode"]:::payload
    E --> F["View page source<br/>full base64 string"]:::payload
    F --> G["Decode base64<br/>configure.php source"]:::user
    G --> H["DB credentials recovered<br/>root : HTB{...}"]:::user

    classDef entry fill:#1d4ed8,stroke:#1e3a8a,color:#ffffff;
    classDef payload fill:#be123c,stroke:#881337,color:#ffffff;
    classDef user fill:#15803d,stroke:#14532d,color:#ffffff;
    linkStyle default stroke-width:2px
```

---

## What I Learned

- How to use ffuf to enumerate hidden `.php` pages on the target and spot `configure.php`.
- Why including `configure.php` directly returns nothing, since PHP executes it instead of printing its source.
- Applying the `php://filter` wrapper with `convert.base64-encode` to read a file as data rather than run it.
- Checking the page source to grab the complete base64 string when the rendered page only shows part of it.
- Decoding the base64 to recover the `configure.php` source and the database credentials inside it.

---

## Enumerating the Target

We can start by navigating to our target web application.

![Inlane Freight homepage loaded in the browser](images/01-target-homepage.png)

*Figure 1 - The target homepage, the same Inlane Freight application seen in earlier labs.*

We can start by using `ffuf` to enumerate our webpage and see the hidden PHP pages. We can use the following payload:

```
ffuf -w /opt/useful/seclists/Discovery/Web-Content/common.txt -u http://154.57.164.75:30373/FUZZ.php
```

![ffuf output listing discovered pages including configure](images/02-ffuf-configure-found.png)

*Figure 2 - ffuf identifies a `configure` page (HTTP 302), i.e. `configure.php`.*

---

## Reading the Source with a PHP Filter

We have identified there is a `configure.php`. We can now use our base64 PHP filter to reveal the information inside it. For that we will add this to the parameter in our URL:

```
index.php?language=php://filter/read=convert.base64-encode/resource=configure
```

![Browser URL bar containing the php filter payload](images/03-filter-payload-url.png)

*Figure 3 - The `php://filter` wrapper with `convert.base64-encode` set on the `language` parameter.*

We can see our output down below.

![Base64 string rendered in the History card on the page](images/04-base64-on-page.png)

*Figure 4 - The file is returned as a base64 string instead of being executed.*

But this isn't the full base64, so we can view the source code to see everything.

![Page source showing the complete base64 string](images/05-view-source-full-base64.png)

*Figure 5 - Viewing the page source (`view-source:`) exposes the complete base64 blob, which the rendered page had cut off.*

---

## Decoding the Credentials

Now we can use a base64 decode tool online to get the password, as this exercise instructs us to.

![base64decode.org showing the decoded configure.php with database credentials](images/06-base64-decoded-creds.png)

*Figure 6 - Decoding the base64 reveals the `configure.php` source, including the database username and password.*

The decoded configuration shows the database username `root` and the password below.

> **Objective: reveal the database username and password.**

**Username:** `root`

**Password:** `HTB{n3v3r_$t0r3_pl4!nt3xt_cr3d$}`
