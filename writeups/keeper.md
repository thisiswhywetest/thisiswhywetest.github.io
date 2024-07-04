---
title: "HTB Keeper"
permalink: /writeups/keeper
location: default
---

# HTB Keeper

**Machine Information**

	IP : 10.10.11.227
	OS : Linux

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.10.98 -- -sC -sV
```

 This should reveal three open ports: `21`, `23`, and `80`.

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

Using the software string at the bottom of the page we can find default credentials during our research.

```
RT 4.4.4+dfsg-2ubuntu1
```

We can login with the credentials `root:password` which are a success. Checking the users shows a default password for one of the users.

<http://tickets.keeper.htb/rt/Admin/Users/Modify.html?id=27>

We can use these credentials to SSH to the target machine.

```
ssh lnorgaard@keeper.htb
```

Grab the flag.

```
cat user.txt
```

### Privilege Escalation

There is a zip file called `RT30000.zip` that we can transfer to our target machine. Attempt to crack the password yields no results so we can try the following exploit:

<https://github.com/matro7sh/keepass-dump-masterkey>

It returns a number of possible passwords. If we search the partial string in Google we get the following result:

<https://www.danishfoodlovers.com/danish-red-berry-pudding/>

Using the dessert name `rødgrød med fløde` we can access the KeePass database.

```
kpcli --kdb passcodes.kdbx
```

There is a `PuTTY-User-Key` file that we want to retrieve.

```
cd /passcodes/Network
ls
show -f 0
```

Copy and paste the entire "Notes" section into a file called `keeper.ppk`. Using putty tools we can split this in an SSH key.

```
puttygen keeper.ppk -O private-openssh -o id_rsa
```

Log in as `root` and grab the flag.
