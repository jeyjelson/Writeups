# Security Lab & CTF Write-Ups
This repo has write-ups from boxes and labs on Hack the Box, HTB Academy,
 PortSwigger, and TryHackMe, plus real engagements I've done outside of labs. It
  covers offensive security work like web app exploitation, injection attacks,
  and breaking authentication, alongside defensive security and some AI security
  too. It's basically a running log of what I've learned and how I approach a
  target.

  Every write-up follows roughly the same format: how I found and figured out the
  target, where the actual weakness was, how I got past whatever was blocking
  me, and how I turned that into real access or proof of impact. Each one has
  screenshots and a diagram of the attack path, so it's easy to follow the
  reasoning even if you weren't the one doing it.

## Offensive Security

### Cross-Site Scripting (XSS)
| Box | Platform | Difficulty | Key techniques |
|-----|----------|------------|----------------|
| [XSS Phishing](./HTB%20XSS%20Phishing%20CTF%20writeup/README.md) | HTB Academy | Intermediate | Cross-site scripting, phishing form injection, credential harvesting |
| [Session Hijacking](./HTB%20Session%20Hijacking%20CTF%20writeup/README.md) | HTB Academy | Intermediate | Blind XSS, cookie theft, session hijacking |
| [XSS Skills Assessment](./HTB%20XSS%20Skills%20Assessment%20CTF%20writeup/README.md) | HTB Academy | Intermediate | Blind XSS, cookie theft, session hijacking |
| [Headless](./HTB%20Headless%20CTF%20Writeup/README.md) | HTB Labs | Easy | Header-based XSS, blind XSS cookie theft, command injection, relative-path privilege escalation |

### Server-Side Request Forgery (SSRF)
| Box | Platform | Difficulty | Key techniques |
|-----|----------|------------|----------------|
| [In-Band SSRF Skills Assessment](./HTB%20In-Band%20SSRF%20Skills%20Assessment%20Writeup/README.md) | HTB Academy | Easy | SSRF, internal port scanning with ffuf, localhost trust abuse |
| [Basic SSRF Against Another Back-End System](./PortSwigger%20Basic%20SSRF%20Against%20Another%20Back-End%20System%20Lab%20Writeup/README.md) | PortSwigger | Apprentice | SSRF, Burp Intruder, internal network scanning, Repeater |
| [SSRF with Blacklist-Based Input Filter](./PortSwigger%20SSRF%20with%20Blacklist-Based%20Input%20Filter%20Lab%20Writeup/README.md) | PortSwigger | Practitioner | SSRF, Burp Repeater, IP obfuscation (127.1 / decimal / octal), case-variation filter bypass |
| [SSRF with Whitelist-Based Input Filter](./PortSwigger%20SSRF%20with%20Whitelist-Based%20Input%20Filter%20Lab%20Writeup/README.md) | PortSwigger | Expert | SSRF, Burp Repeater, URL parser confusion, embedded credentials (user@host), fragment (#) smuggling, double URL encoding (%2523) |
| [SSRF Enumeration Skills Assessment](./HTB%20SSRF%20Enumeration%20Skills%20Assessment%20Writeup/README.md) | HTB Academy | Easy | SSRF directory enumeration with ffuf, Apache error filtering, internal admin access |
| [Blind SSRF with Out-of-Band Detection](./PortSwigger%20Blind%20SSRF%20with%20Out-of-Band%20Detection%20Lab%20Writeup/README.md) | PortSwigger Web Security Academy | Practitioner | Blind SSRF, Burp Collaborator, OAST, Referer header injection |
| [Blind SSRF Enumeration](./HTB%20Blind%20SSRF%20Skills%20Assessment%20Writeup/README.md) | HTB Academy | Easy | Blind SSRF, dateserver parameter abuse, ffuf port fuzzing, internal loopback enumeration |
| [SSRF with Filter Bypass via Open Redirection](./PortSwigger%20SSRF%20with%20Filter%20Bypass%20via%20Open%20Redirection%20Lab%20Writeup/README.md) | PortSwigger | Practitioner | SSRF, Burp Repeater, open redirect discovery, redirect chaining, internal admin access |
| [Server-Side Attacks Skills Assessment](./HTB%20Server-Side%20Attacks%20Skills%20Assessment%20Writeup/README.md) | HTB Academy| Intermediate | SSRF, SSTI, Twig RCE, ffuf, Burp Suite |



### Injection
| Box | Platform | Difficulty | Key techniques |
|-----|----------|------------|----------------|
| [Recruit](./THM%20Recruit%20CTF%20Writeup/README.md) | TryHackMe | Intermediate | Enumeration, LFI, SQL injection |
| [Validation](./HTB%20Validation%20CTF%20writeup/README.md) | HTB Labs | Easy | SQL injection, web shell, privilege escalation |
| [Command Injection Skills Assessment](./HTB%20Command%20Injection%20Skills%20Assessment%20Writeup/README.md) | HTB Academy | Easy | Command injection, filter bypass, ${IFS} and ${PATH:0:1} obfuscation, path traversal |
| [Identifying Template Engine for SSTI](./HTB%20SSTI%20Template%20Engine%20Identification%20Writeup/README.md) | HTB Academy | Easy | SSTI, template engine fingerprinting, decision-tree payload testing, Twig vs Jinja2 identification |
| [SSTI Exploitation Jinja2 Flask](./HTB%20SSTI%20Exploitation%20Jinja2%20Flask%20CTF%20Writeup/README.md) | HTB Academy | Easy | Jinja2 SSTI, config.items disclosure, __builtins__ enumeration, LFI via open, os.popen RCE |
| [SSTI Exploitation Twig](./HTB%20SSTI%20Exploitation%20Twig%20CTF%20Writeup/README.md) | HTB Academy | Easy | Twig SSTI, _self enumeration, Symfony file_excerpt LFI, filter+system RCE, path traversal |
| [Identifying and Exploiting Server-Side Includes Injection](./HTB%20Identifying%20and%20Exploiting%20Server-Side%20Includes%20Injection%20CTF%20Writeup/README.md) | HTB Academy| Easy | SSI injection, .shtml detection, #printenv confirmation, #exec RCE, path traversal |
| [Identifying and Exploiting an XSLT Injection](./HTB%20Identifying%20and%20Exploiting%20an%20XSLT%20Injection%20CTF%20Writeup/README.md) | HTB Academy | Easy | XSLT injection, system-property fingerprinting, libxslt, php:function, file_get_contents LFI, system() RCE |

### File Upload Attacks
| Box | Platform | Difficulty | Key techniques |
|-----|----------|------------|----------------|
| [File Upload Exploitation Using a Web Shell](./HTB%20File%20Upload%20Exploitation%20Using%20a%20Web%20Shell%20Lab%20Writeup/README.md) | HTB Academy Academy | Easy | File upload, PHP web shell, msfvenom, backend fingerprinting, path traversal |
| [Bypassing Client-Side Validation for File Upload Attacks](./HTB%20Bypassing%20Client-Side%20Validation%20for%20File%20Upload%20Attacks%20CTF%20Writeup/README.md) | HTB Academy | Easy | Burp Suite, Repeater, PHP web shell, client-side validation bypass, browser dev tools |
| [Bypassing Blacklisted File Upload Filters](./HTB%20Bypassing%20Blacklisted%20File%20Upload%20Filters%20CTF%20Writeup/README.md) | HTB Academy | Easy | File upload blacklist bypass, Burp Intruder extension fuzzing, PHP webshell, RCE |
| [Bypassing Whitelist Filters for File Upload Attacks](./HTB%20Bypassing%20Whitelist%20Filters%20for%20File%20Upload%20Attacks%20CTF%20Writeup/README.md) | HTB Academy | Medium | File upload bypass, blacklist/whitelist evasion, reverse double extension, Burp Intruder fuzzing, PHP webshell |
| [Bypassing Type Filters for File Upload Attacks](./HTB%20Bypassing%20Type%20Filters%20for%20File%20Upload%20Attacks%20CTF%20Writeup/README.md) | HTB Academy | Medium | Burp Intruder, extension enumeration, GIF8 magic bytes, MIME/Content-Type spoofing, reverse double extension, RCE |
| [File Upload via Path Traversal](./PortSwigger%20File%20Upload%20via%20Path%20Traversal%20CTF%20Writeup/README.md) | PortSwigger | Practitioner | File upload bypass, Path traversal, URL encoding, Burp Repeater, PHP web shell, RCE |
| [XXE via SVG Upload Against a Secure File Server](./HTB%20XXE%20via%20SVG%20Upload%20Against%20a%20Secure%20File%20Server%20CTF%20Writeup/README.md) | HTB Academy | Easy | Extension enumeration, XXE, SVG external entity, file:// LFI, php://filter base64, source disclosure |
| [File Upload Attacks Skills Assessment](./HTB%20File%20Upload%20Attacks%20Skills%20Assessment%20Writeup/README.md) | HTB Academy | Medium | File upload bypass, Burp Intruder, client-side validation bypass, SVG XXE, php://filter source disclosure, RCE |

## Broken Authentication
| Box | Platform | Difficulty | Key techniques |
|-----|----------|------------|----------------|
| [Username Enumeration](./HTB%20Username%20Enumeration%20CTF%20Writeup/README.md) | HTB Academy | Easy | Username enumeration, ffuf, Burp Suite, error-message analysis |
| [Brute Forcing Passwords](./HTB%20Brute%20Forcing%20Passwords%20CTF%20Writeup/README.md) | HTB Academy | Easy | Password brute-forcing, ffuf, custom wordlist, Burp Suite |
| [Brute Forcing 2FA Codes](./HTB%20Brute%20Forcing%202FA%20Codes%20CTF%20Writeup/README.md) | HTB Academy | Easy | 2FA/OTP brute-forcing, ffuf, session cookie, Burp Suite |
| [Direct Access via Broken Authentication](./HTB%20Direct%20Access%20via%20Broken%20Authentication%20CTF%20Writeup/README.md) | HTB Academy | Easy | Burp Suite response interception, HTTP 302 to 200 tampering, broken access control |
| [Authentication Bypass via Parameter Modification (IDOR)](./HTB%20Authentication%20Bypass%20via%20Parameter%20Modification%20IDOR%20CTF%20Writeup/README.md) | HTB Academy | Easy | IDOR / broken access control, Burp Intruder enumeration, ffuf fuzzing, seq wordlist generation |
| [Attacking Session Tokens](./HTB%20Attacking%20Session%20Tokens%20CTF%20Writeup/README.md) | HTB Academy | Easy | Session token tampering, hex decode/encode with xxd, cookie forgery, privilege escalation |
| [Broken Authentication Skills Assessment](./HTB%20Broken%20Authentication%20Skills%20Assessment%20Writeup/README.md) | HTB Academy Academy | Medium | Username enumeration, ffuf brute forcing, custom wordlist, 2FA bypass, Burp response interception |

## File Inclusion
| Box | Platform | Difficulty | Key techniques |
|-----|----------|------------|----------------|
| [Local File Inclusion](./HTB%20Local%20File%20Inclusion%20CTF%20Writeup/README.md) | HTB Academy | Easy | LFI, path traversal, /etc/passwd user enumeration |
| [Bypassing Multiple Filters](./HTB%20Bypassing%20Multiple%20Filters%20CTF%20Writeup/README.md) | HTB Academy | Easy | LFI, filter bypass, ....// traversal, approved-path prefix |
| [PHP Filters: Source Code Disclosure](./HTB%20PHP%20Filters%20Source%20Code%20Disclosure%20CTF%20Writeup/README.md) | Hack The Box | Easy | LFI, php://filter, base64 source disclosure, ffuf |


## Defensive Security
| Box | Platform | Difficulty | Key techniques |
|-----|----------|------------|----------------|
| [Incident Handling](./HTB%20Incident%20Handling%20Write-up/README.md) | HTB Academy | Easy | TheHive triage, VirusTotal enrichment, MITRE ATT&CK mapping, Base64 PowerShell decoding |
| [Nessus Vulnerability Assessment](./HTB%20Nessus%20Vulnerability%20Assessment/README.md) | HTB Academy | Easy | Authenticated vulnerability scanning |

## AI Security
| Box | Platform | Difficulty | Key techniques |
|-----|----------|------------|----------------|
| [ContAInment](./ContAInment%20THM%20CTF%20writeup/README.md) | TryHackMe | Intermediate | Phishing analysis, PCAP forensics, prompt injection, LLM exploitation |

