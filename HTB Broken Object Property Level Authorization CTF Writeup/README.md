# Hack The Box Academy - Broken Object Property Level Authorization | Write-up

> **Platform:** Hack The Box Academy &nbsp;•&nbsp; **Category:** API / Web &nbsp;•&nbsp; **Difficulty:** Easy
>
> **Target:** `154.57.164.82:32057` &nbsp;•&nbsp; **Time taken:** 20 mins
>
> **Author:** Jithin Jelson

---

## Introduction

In this exercise we are told that there is Broken Object Property Level Authorization. Since this is a category of vulnerabilities, it encompasses two subclasses, Excessive Data Exposure and Mass Assignment. Therefore in this exercise we have to find a flag for each of the vulnerabilities.

Credentials given: `htbpentester5@hackthebox.com:HTBPentester5` (and a second account for the Mass Assignment stage `htbpentester7@hackthebox.com:HTBPentester7`).

---

## Assessment Overview

```mermaid
flowchart LR
    A[Account 1<br/>sign in<br/>obtain JWT]:::entry --> B[roles: supplier<br/>read access]:::intel
    B --> C[GET suppliers<br/>emails and phone<br/>numbers exposed]:::mitre
    C --> D[GET supplier-companies<br/>Flag 1<br/>Excessive Data Exposure]:::user

    E[Account 2<br/>sign in<br/>obtain JWT]:::entry --> F[roles: order<br/>create access]:::intel
    F --> G[GET products<br/>grab a valid<br/>ProductID]:::intel
    G --> H[Create order then<br/>POST order items<br/>Quantity and NetSum<br/>user-controlled]:::payload
    H --> I[Flag 2<br/>Mass Assignment]:::user

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

- How Broken Object Property Level Authorization splits into two distinct flaws, Excessive Data Exposure and Mass Assignment.
- Reading a user's roles first to decide which endpoints are worth visiting one by one.
- Spotting Excessive Data Exposure when a bulk endpoint returns fields like email and phone number that the caller should never see.
- Recognising that sensitive data, and even a flag, can be hidden inside an ordinary-looking field of a bulk response.
- Abusing Mass Assignment by supplying user-controlled fields such as Quantity and NetSum that the server binds without validation.
- Chaining a valid ProductID from an unrestricted products endpoint into an order-item creation request to trigger the flaw.

---

# Part 1 — Excessive Data Exposure

## Enumerating Our Access

We can start by trying to find our Excessive Data Exposure vulnerability. We head to our target IP with `/swagger/index.html` and authorise ourselves with our JWT token.

Now when we head down to roles we can see what is available to us.

![roles/current-user for the first account](images/01-roles-account1.png)

*Figure 1 - Our roles: Suppliers_Get, Suppliers_GetAll, SupplierCompanies_Get, and SupplierCompanies_GetAll*

## Finding the Exposed Data

So we can visit these one by one to find our flag. We can see our vulnerability here, as we are able to view the email and phone number of all the suppliers.

![Suppliers list exposing emails and phone numbers](images/02-suppliers-pii.png)

*Figure 2 - The suppliers endpoint leaks every supplier's email and phone number*

And we found our flag in the supplier companies endpoint `/api/v1/supplier-companies`.

![Flag hidden in the supplier-companies response](images/03-flag1-supplier-companies.png)

*Figure 3 - The "HTB Academy" supplier company's email field contains the flag*

> **What is the Excessive Data Exposure flag?**

**Answer:** `HTB{d759c70b5a9f6a392af78cc1eca9cdf0}`

---

# Part 2 — Mass Assignment

## Logging In With the Second Account

Now for our second vulnerability we have to log in with the second set of credentials provided to us. This time we have the following roles that we are allowed to access.

![roles/current-user for the second account](images/04-roles-account2.png)

*Figure 4 - Our roles: CustomerOrders_GetByID, CustomerOrders_Create, CustomerOrderItems_Get, and CustomerOrderItems_Create*

## Building the Malicious Order

We will start by creating a new customer order, and also grab our own product in all products.

![Products list to obtain a valid ProductID](images/05-products-list.png)

*Figure 5 - The unrestricted products endpoint gives us a valid ProductID*

Now we can put these two together to create a new order item, supplying the OrderID and ProductID along with a Quantity and NetSum of our choosing.

![Creating an order item with user-controlled fields](images/06-create-order-item.png)

*Figure 6 - Posting an order item with a NetSum we control*

And when we execute we get our flag.

![Flag returned in the order-item creation response](images/07-flag2-mass-assignment.png)

*Figure 7 - The response returns SuccessStatus true and the flag in the Message field*

> **What is the Mass Assignment flag?**

**Answer:** `HTB{4d86794f82046e465ca29d91bdbe5bca}`

## Why It Works

Here's the problem. The API lets us control:

- OrderID
- ProductID
- Quantity
- NetSum

No validation. No restrictions. That's Mass Assignment.
