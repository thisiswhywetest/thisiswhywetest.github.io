---
title: "HTB Delivery"
permalink: /writeups/delivery
location: default
---

# HTB Delivery

**Machine Information**

	IP : 10.10.10.222
	OS : Linux

### Reconnaissance

**Rustscan**

Start by running `rustscan` to see which ports are open.

```
rustscan -a 10.10.10.222 -- -sC -sV
```

This scan found three open ports: `22`, `80`, and `8065`.

```
PORT     STATE SERVICE REASON  VERSION
22/tcp   open  ssh     syn-ack OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 9c:40:fa:85:9b:01:ac:ac:0e:bc:0c:19:51:8a:ee:27 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCq549E025Q9FR27LDR6WZRQ52ikKjKUQLmE9ndEKjB0i1qOoL+WzkvqTdqEU6fFW6AqUIdSEd7GMNSMOk66otFgSoerK6MmH5IZjy4JqMoNVPDdWfmEiagBlG3H7IZ7yAO8gcg0RRrIQjE7XTMV09GmxEUtjojoLoqudUvbUi8COHCO6baVmyjZRlXRCQ6qTKIxRZbUAo0GOY8bYmf9sMLf70w6u/xbE2EYDFH+w60ES2K906x7lyfEPe73NfAIEhHNL8DBAUfQWzQjVjYNOLqGp/WdlKA1RLAOklpIdJQ9iehsH0q6nqjeTUv47mIHUiqaM+vlkCEAN3AAQH5mB/1
|   256 5a:0c:c0:3b:9b:76:55:2e:6e:c4:f4:b9:5d:76:17:09 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBAiAKnk2lw0GxzzqMXNsPQ1bTk35WwxCa3ED5H34T1yYMiXnRlfssJwso60D34/IM8vYXH0rznR9tHvjdN7R3hY=
|   256 b7:9d:f7:48:9d:a2:f2:76:30:fd:42:d3:35:3a:80:8c (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEV5D6eYjySqfhW4l4IF1SZkZHxIRihnY6Mn6D8mLEW7
80/tcp   open  http    syn-ack nginx 1.14.2
| http-methods:
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.14.2
|_http-title: Welcome
8065/tcp open  unknown syn-ack
| fingerprint-strings:
|   GenericLines, Help, RTSPRequest, SSLSessionReq, TerminalServerCookie:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest:
|     HTTP/1.0 200 OK
|     Accept-Ranges: bytes
|     Cache-Control: no-cache, max-age=31556926, public
|     Content-Length: 3108
|     Content-Security-Policy: frame-ancestors 'self'; script-src 'self' cdn.rudderlabs.com
|     Content-Type: text/html; charset=utf-8
|     Last-Modified: Tue, 02 Jul 2024 05:11:51 GMT
|     X-Frame-Options: SAMEORIGIN
|     X-Request-Id: ebbkmwo3g78rdfiagkhnzar8ze
|     X-Version-Id: 5.30.0.5.30.1.57fb31b889bf81d99d8af8176d4bbaaa.false
|     Date: Tue, 02 Jul 2024 05:13:19 GMT
|     <!doctype html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=0"><meta name="robots" content="noindex, nofollow"><meta name="referrer" content="no-referrer"><title>Mattermost</title><meta name="mobile-web-app-capable" content="yes"><meta name="application-name" content="Mattermost"><meta name="format-detection" content="telephone=no"><link re
|   HTTPOptions:
|     HTTP/1.0 405 Method Not Allowed
|     Date: Tue, 02 Jul 2024 05:13:19 GMT
|_    Content-Length: 0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Getting a Foothold

**Update /etc/hosts**

Exploring the website will find the subdomain `helpdesk.delivery.htb` so make sure to update the `/etc/hosts` file to include both the domain and subdomain.

**Create an account**

After updating the `/etc/hosts` file we will be able to access the ticketing system. There is a function to create an account but I had no success with this.

<http://helpdesk.delivery.htb/account.php?do=create>

**Lodge a ticket**

Next we can try to create a new ticket using the following URL:

<http://helpdesk.delivery.htb/open.php>

It is important to use an email address that does not exist as the ticketing system creates an account for us. Upon completion we get the following message:

```
Null, 

You may check the status of your ticket, by navigating to the Check Status page using ticket id: 4209048.

If you want to add more information to your ticket, just email 4209048@delivery.htb.

Thanks,

Support Team
```

**Check the ticket status**

Using the email address we provided and our ticket ID we can see the status of the ticket. This has create a temporary inbox that we can use to create an account with Mattermost on port 8065. Use the email address previously disclosed and create an account. If we check the status of the ticket again we will see the confirmation email. After clicking this link (might need to copy and paste) we can sign into the account using the password we provided. The Mattermost channel will reveal the following credentials for the user `maildeliverer` in clear text.

It also has a hint about passwords being re-used:

> Also please create a program to help us stop re-using the same passwords everywhere.... Especially those that are a variant of "PleaseSubscribe!"

SSH to the target and server and get the flag.

### Privilege Escalation

Check the `/etc/passwd` file shows a `mysql` user indicating there is likely a database in use. If we check the configuration files for `mattermost` we find credentials for the `mmuser` user.

```
cat /opt/mattermost/config/config.json
```

**MySQL**

Connect to the MySQL database using the credentials we found previously.

```
mysql -u mmuser -p
```

Investigate the tables to find a password hash for the `root` user.

```
show databases;
use mattermost;
show tables;
describe Users;
select Username,Password from Users;
```

Using the hint we found earlier we can use a wordlist that just contains `PleaseSubscribe!` and a `hashcat` rule to create variations of this password. Using `hashcat` we can crack the root hash.

```
hashcat -m 3200 hash password -r /usr/share/hashcat/rules/best64.rule
```

Switch to the `root` user and grab the flag.

```
su - root
```
