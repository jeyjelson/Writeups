# Hack The Box - Attacking Session Tokens | Write-up

> **Platform:** Hack The Box &nbsp;•&nbsp; **Category:** Web Exploitation / Session Management &nbsp;•&nbsp; **Difficulty:** Easy
>
> **Target:** `154.57.164.82:31710` &nbsp;•&nbsp; **Time taken:** 15 minutes
>
> **Author:** Jithin Jelson

---

## Introduction

We have been told in this instance that our web application has a flaw in its token handler. We have been told that it is predictable. Based on what we find out we must either brute force or guess the next session token.

**Credentials provided:** `htb-stdnt` / `AcademyStudent!`

---

## Assessment Overview

```mermaid
flowchart LR
    A[Login as htb-stdnt<br/>AcademyStudent!]:::entry --> B[Intercept /admin.php<br/>in Burp Proxy]:::entry
    B --> C[session cookie<br/>long hex string]:::intel
    C --> D[Re-login check<br/>token stays the same]:::ioc
    C --> E[Decode with xxd -r -p<br/>user=htb-stdnt;role=user]:::payload
    E --> F[Forge role=admin<br/>re-encode with xxd -p]:::payload
    F --> G[Swap in the new<br/>session cookie]:::payload
    G --> H[Admin dashboard<br/>HTB flag revealed]:::user

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

- How a session token that is just hex-encoded plaintext offers no real protection.
- Decoding a cookie with `xxd -r -p` to reveal the fields hidden inside it.
- Spotting that a token never changes between logins, which hints it is not random.
- Forging a new token by editing `role=user` to `role=admin` and re-encoding with `xxd -p`.
- Swapping the crafted cookie back into the request to escalate from user to admin.
- That encoding is not encryption, so anything reversible is a data tampering risk.

---

## Logging In

We can start by visiting our web application, loading up Burp Suite and entering our given credentials.

![Burp Proxy intercepting the request to admin.php](images/01-login-intercept.png)

*Figure 1 - Intercepting `GET /admin.php` in Burp Proxy. The request carries a `session=` cookie, and the page shows "Could not load admin data. Please check your privileges".*

---

## Capturing the Session Cookie

Once we intercepted the request we are able to see our cookie session. Let's take note of this.

```
757365723d6874622d7374646e743b726f6c653d75736572
```

We can log out and log in again to see if there is any changes, although it looks like hex encoded. It looks like it is the exact same.

![Session cookie value in the Burp Inspector](images/02-session-cookie.png)

*Figure 2 - The `session` cookie value. After logging out and back in, the token is identical, which suggests it is not random.*

---

## Decoding the Token

Let's use xxd to decode it.

```
echo -n 757365723d6874622d7374646e743b726f6c653d75736572 | xxd -r -p
```

![Decoded token showing user=htb-stdnt;role=user](images/03-decode-token.png)

*Figure 3 - The hex decodes to `user=htb-stdnt;role=user`, so the token is just readable key-value plaintext.*

---

## Forging an Admin Token

Now let's try and forge this to role as admin.

```
echo -n 'user=htb-stdnt;role=admin' | xxd -p
```

![Re-encoding the forged admin token to hex](images/04-forge-admin-token.png)

*Figure 4 - Changing `role=user` to `role=admin` and re-encoding it back to hex with `xxd -p`.*

---

## Session Takeover

Now we can change our session id to this, and it worked.

![Admin dashboard revealing the flag](images/05-admin-panel-flag.png)

*Figure 5 - Swapping in the forged cookie loads the full admin dashboard and reveals the flag.*

> Forge an admin session token and retrieve the flag.

**Answer:** `HTB{d1f5d760d130f7dd11de93f0b393abda}`
