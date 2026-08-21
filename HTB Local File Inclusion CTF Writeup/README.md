# Hack The Box - Local File Inclusion | Write-up

> **Platform:** Hack The Box &nbsp;•&nbsp; **Category:** Web / File Inclusion &nbsp;•&nbsp; **Difficulty:** Easy
>
> **Target:** `154.57.164.82:32421` &nbsp;•&nbsp; **Time taken:** 5 mins
>
> **Author:** Jithin Jelson

---

## Introduction

In this exercise we are told that our web application is vulnerable to local file inclusion. We are not told if there are any defensive measures put in place, but the web application has errors in place for us to learn from and change our payload. In a real life scenario the errors should never be verbose.

---

## Assessment Overview

```mermaid
flowchart LR
    A["Browse target<br/>Inlane Freight"]:::entry --> B["Language parameter<br/>index.php?language="]:::entry
    B --> C["Basic LFI attempt<br/>language=/etc/passwd<br/>nothing loads"]:::payload
    C --> D["Path traversal<br/>../../../../etc/passwd"]:::payload
    D --> E["/etc/passwd disclosed<br/>enumerate system users"]:::user
    E --> F["User found: barry"]:::user
    D --> G["Traversal to flag<br/>usr/share/flags/flag.txt"]:::payload
    G --> H["Flag captured<br/>HTB{...}"]:::user

    classDef entry fill:#1d4ed8,stroke:#1e3a8a,color:#ffffff;
    classDef payload fill:#be123c,stroke:#881337,color:#ffffff;
    classDef user fill:#15803d,stroke:#14532d,color:#ffffff;
    linkStyle default stroke-width:2px
```

---

## What I Learned

- How the `language` parameter controls which file the page loads, which is exactly what makes the LFI possible.
- Why a plain `/etc/passwd` payload failed while a path traversal payload succeeded.
- Using `../../../../` to climb to the filesystem root before naming the file I actually wanted.
- Reading `/etc/passwd` to enumerate the users on the system, which gave up the account `barry`.
- Reaching outside the web root to pull `/usr/share/flags/flag.txt` and capture the flag.
- That the verbose errors, left on here for learning, would be turned off in a real deployment so an attacker gets no such hints.

---

## Enumerating the Language Parameter

We can start by loading up our browser and navigating to our target IP address.

![Inlane Freight homepage loaded in the browser](images/01-target-homepage.png)

*Figure 1 - The target homepage (Inlane Freight) loaded over HTTP.*

When we select the language option we get two options, English or Spanish, and when we click Spanish we can see our URL parameter has changed.

![Spanish content loaded with language parameter visible in the URL](images/02-language-spanish-param.png)

*Figure 2 - Selecting Spanish changes the page content and sets `index.php?language=es.php` in the URL.*

---

## Basic LFI and Path Traversal

We will attempt a basic LFI here by just typing `/etc/passwd` after the language parameter.

```
index.php?language=/etc/passwd
```

![Basic LFI attempt with /etc/passwd, page content missing](images/03-basic-lfi-etc-passwd.png)

*Figure 3 - The `/etc/passwd` payload on its own, the page content does not load.*

It seems like nothing has loaded, perhaps we need to do some path traversal so we can go to the root by doing `../../../../etc/passwd`.

```
index.php?language=../../../../etc/passwd
```

![Path traversal payload returning the full contents of /etc/passwd](images/04-path-traversal-passwd.png)

*Figure 4 - The path traversal payload works and loads our specified file, the full `/etc/passwd`.*

Now it seems to load our specified file. With this information we can answer our first question.

> **Using the file inclusion, find the name of a user on the system that starts with "b".**

We can see the answer to this is `barry`.

![Close-up of the barry entry in /etc/passwd](images/05-user-barry.png)

*Figure 5 - The `barry:x:1000:1000::/home/barry:/bin/sh` line confirms the user.*

**Answer:** `barry`

---

## Capturing the Flag

Now the second part of the question asks us:

> **Submit the contents of the flag.txt file located in the /usr/share/flags directory.**

So let's replace `etc/passwd` with `/../../../../../../usr/share/flags/flag.txt`.

```
index.php?language=/../../../../../../usr/share/flags/flag.txt
```

![Flag contents displayed on the page after traversal to flag.txt](images/06-flag-captured.png)

*Figure 6 - The traversal payload reads `flag.txt` and prints the flag on the page.*

**Answer:** `HTB{n3v3r_tru$t_u$3r_!nput}`
