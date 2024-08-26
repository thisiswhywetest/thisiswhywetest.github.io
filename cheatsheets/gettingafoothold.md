# Getting a Foothold

### Table of Contents

0. [Scan all the Ports](#scan-all-the-ports)
1. [Update Hosts File](#Update-Hosts-File)
2. [Enumerate, Enumerate, Enumerate](#enumerate,-enumerate,-enumerate)

## Scan all the Ports

**Rustscan**

```
rustscan -a 127.0.0.1 -- -sC -sV
```

**Nmap (Linux targets)**

```
nmap -p- -sC -sV 127.0.0.1
```

**Nmap (Windows targets)**

```
nmap -p- -Pn -sC -sV 127.0.0.1
```

## Update Hosts File

```
sudo vim /etc/hosts
```

Add a line similar to the following:

```
127.0.0.1 domain.local
```

## Enumerate, Enumerate, Enumerate

### FTP - 21/tcp

**Anonymous login**

```
ftp anonymous@127.0.0.1
```

**Check default credentials**

```
hydra -C /usr/share/wordlists/seclists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt ftp://127.0.0.1
```

**Brute force with valid usernames**

```
hydra -L usernames.txt -P /usr/share/wordlists/rockyou.txt ftp://127.0.0.1
```

**Download everything**

```
wget -r ftp://127.0.0.1
```

### SSH - 22/tcp

**Check for outdated version**

```
nmap -p 22 -sV 127.0.0.1
```

### SMTP - 25/tcp, 465/tcp, 587/tcp

**Manually check valid usernames**

```
telnet 127.0.0.1 25
```

```
VRFY username
```

**SMTP user enumeration tool**

```
smtp-user-enum -M RCPT -U /usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt -D domain.local -t 127.0.0.1
```

**SMTP open relay**

```
nmap -p 25 --script smtp-open-relay 127.0.0.1
```

### DNS - 53/tcp

**Zone transfer**

```
dig axfr @127.0.0.1 domain.local
```

### HTTP - 80/tcp, 443/tcp

#### Virtual Host Enumeration

**FFuF**

```
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://FUZZ.domain.local
```

**Wfuzz**

```
wfuzz -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://domain.local -H 'Host:FUZZ.domain.local'
```

**NOTE:** add the parameter `--hw` to remove false positives

#### Directory Enumeration

**Dirsearch**

```
dirsearch -u http://127.0.0.1
```

**Feroxbuster**

```
feroxbuster --url http://127.0.0.1
```

**Gobuster**

```
gobuster dir -u http://127.0.0.1 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

#### Common Files

Check for common files and directories on the webserver such as:
- `robots.txt`
- `sitemap.xml`
- `index.php`
- `/admin`
- `/wp-admin`

### RPC - 111/tcp

**RPC Information**

```
rpcinfo 127.0.0.1
```

### SMB - 139/tcp, 445/tcp

**SMB Map**

```
smbmap -H 127.0.0.1
```

**List shares with anonymous access enabled**

```
smbclient -N -L 127.0.0.1
```

**Connect to shares with anonymous access enabled**

```
smbclient -N \\127.0.0.1\share_name
```

**Mount share**

```
mkdir /tmp/smb
```

```
mount -t cifs //127.0.0.1/share_name /tmp/smb
```

```
ls -la /tmp/smb
```

### IMAP - 143/tcp, 993/tcp



### Rsync - 873/tcp

**Nmap**

```
nmap -p 873 -sC -sV 127.0.0.1
```

**Probe for shares**

```
nc -nv 127.0.0.1 873
```

**Enumerate open share**

```
rsync -av --list-only rsync://127.0.0.1/share
```

### MSSQL - 1433/tcp

**Nmap**

```
nmap -p 1433 --script ms-sql* 127.0.0.1
```

**Impacket MSSQL client**

```
impacket-mssqlclient username@127.0.0.1 -windows-auth
```

### Oracle TNS - 1521/tcp

**Nmap**

```
nmap -p 1521 --script oracle* 127.0.0.1
```

**ODAT**

```
odat all -s 127.0.0.1
```

**SQLplus**

```
sqlplus username/password@127.0.0.1/sid
```

### NFS - 2049/tcp

**Show available NFS shares**

```
showmount -e 127.0.0.1
```

**Mount NFS share**

```
mkdir /tmp/nfs
```

```
mount -t nfs [o vers=2] 127.0.0.1:/share_name /tmp/nfs -o nolock
```

### MySQL - 3306/tcp

**Nmap**

```
nmap -p 3306 --script mysql* 127.0.0.1
```

**MySQL client**

```
mysql -u username -p -h 127.0.0.1
```

### RDP - 3389/tcp

**Nmap**

```
nmap -p 3389 --script rdp* 127.0.0.1
```

**xFreeRDP**

```
xfreerdp /u:username /p:password /v:127.0.0.1
```

### WinRM - 5985/tcp, 5986/tcp

**Nmap**

```
nmap -p 5985,5986 -sC -sV --disable-arp-ping 127.0.0.1
```

**Connect to WinRM**

```
evil-winrm -i 127.0.0.1 -u username -p password
```

### Redis - 6379/tcp



### SNMP - 161/udp

**SNMPwalk**

```
snmpwalk -v2c -c public 127.0.0.1
```

**OneSixyOne**

```
onesixtyone -c /usr/share/wordlists/seclists/Discovery/SNMP/snmp.txt 127.0.0.1
```

### IPMI - 623/udp

**Nmap**

```
sudo nmap -p 623 -sU --script ipmi* 127.0.0.1
```

**Metasploit**

```
use auxiliary/scanner/ipmi/ipmi_dumphashes
```
