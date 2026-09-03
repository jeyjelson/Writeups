# PortSwigger - SSRF with Filter Bypass via Open Redirection | Write-up

> **Platform:** PortSwigger &nbsp;•&nbsp; **Category:** Web Security / SSRF &nbsp;•&nbsp; **Difficulty:** Practitioner
>
> **Target:** `web (lab instance)` &nbsp;•&nbsp; **Time taken:** 15 mins
>
> **Author:** Jithin Jelson

---

## Introduction

This lab has a stock check feature which fetches data from an internal system. To solve the lab we have been told to delete the user carlos from the admin panel located at 192.168.0.12:8080/admin. We have been told that the stock checker has been restricted to only the local application, so we will need to find an open redirect affecting the application first.

---

## Assessment Overview

```mermaid
flowchart LR
    A[Stock check feature<br/>SSRF, local-only]:::entry

    A --> B[Direct: stockApi to<br/>192.168.0.12:8080/admin]:::payload
    B --> B1[Blocked: Invalid<br/>external stock check url]:::mitre

    A --> C[Probe &path= on<br/>the GET request]:::payload
    C --> C1[Server times out<br/>dead end]:::mitre

    B1 -.-> D{Filter allows only<br/>local app URLs}:::intel
    C1 -.-> D
    A --> E(("Open redirect hub<br/>nextProduct path=")):::intel
    D --> E

    E --> E1[path=google.com<br/>302 confirmed]:::ioc
    E1 --> G[Chain to<br/>192.168.0.12:8080/admin<br/>admin panel reached]:::ioc
    E1 --> H[Chain to /admin/delete<br/>username=carlos]:::payload

    H --> I[User deleted<br/>successfully]:::user
    G --> K[Lab solved]:::user
    I --> K

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

- How an SSRF filter that only allows local application URLs can still be bypassed if the app contains an open redirect.
- Why an open redirect is so useful here, since the server treats the redirecting URL as local but then follows it to an internal host.
- Finding an open redirect by feeding a path parameter an external URL like google.com and watching for a 302 response.
- Chaining the open redirect into the stockApi parameter to reach 192.168.0.12:8080/admin and delete the target user.

---

## Intercepting the Stock Check Feature

First we can start up Burp Suite and load up our target page.

![Stock check request intercepted in Burp](images/01-stock-check-intercept.png)

*Figure 1 - The stock check request captured in Burp, with the `stockApi` parameter pointing at a relative `/product/stock/check` path on the local application.*

---

## Testing Direct Access to the Admin Panel

We test to see if we can access the 192.168.0.12:8080/admin panel directly first to see what response we get.

```
stockApi=http://192.168.0.12:8080/admin
```

![Direct access to the internal admin panel is blocked](images/02-direct-admin-blocked.png)

*Figure 2 - Pointing `stockApi` straight at the internal admin host returns `400 Bad Request` with "Invalid external stock check url 'Invalid URL'", confirming the checker is restricted to the local application.*

---

## Attempting an Open Redirect

We can see that we get an invalid external stock check URL, so we can attempt an open redirect to see if we can bypass this. We will use &path=192.168.0.12:8080/admin.

![Open redirect path parameter attempt](images/03-open-redirect-path-attempt.png)

*Figure 3 - Appending a `path` parameter aimed at the internal admin host, testing whether the application will redirect to it.*

Seems like the vulnerability isn't there, it could be in the GET request to the application. We also came across a next page button that has a path URL, and to test for the open redirect vulnerability we put google and we were successful.

```
GET /product/nextProduct?currentProductId=1&path=http://google.com
```

![Open redirect confirmed with a 302 to google.com](images/04-open-redirect-confirmed-google.png)

*Figure 4 - The `/product/nextProduct` endpoint takes a `path` parameter and responds with `302 Found` and `Location: http://google.com`, so we have a working open redirect.*

---

## Chaining the Open Redirect Into the SSRF

So my next thinking was we can use this URL in our vulnerable URL field in stockApi and try to access the admin panel from there, as it is a valid URL path.

```
stockApi=/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin
```

![Redirect chain reaches the internal admin panel](images/05-redirect-chain-admin-panel.png)

*Figure 5 - Feeding `stockApi` the local `nextProduct` endpoint, which then redirects to `192.168.0.12:8080/admin`, passes the filter and reaches the internal admin panel (note the `/admin` link in the response).*

This time it seemed to work. We tried it initially on the GET request but the server timed out, maybe this was because that area wasn't vulnerable?

---

## Deleting carlos and Solving the Lab

And just like that we had solved the lab.

```
stockApi=/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin/delete?username=carlos
```

**Answer:** `stockApi=/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin/delete?username=carlos`

![User deleted and lab solved](images/06-delete-carlos-solved.png)

*Figure 6 - Extending the redirect target with `/admin/delete?username=carlos` deletes the user, the response confirms "User deleted successfully", and the lab is marked solved.*
