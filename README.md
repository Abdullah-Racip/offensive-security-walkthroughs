<p align="center">
  <img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=20&pause=1200&color=39FF14&center=true&vCenter=true&width=600&lines=whoami%3A+Abdullah;cat+%2Fetc%2Fpasswd+--dreams;Rooting+boxes%2C+one+writeup+at+a+time" alt="Typing SVG" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/boxes_rooted-5-success?style=flat-square" alt="Boxes rooted"/>
  <img src="https://img.shields.io/badge/platform-VulnHub-orange?style=flat-square" alt="Platform"/>
  <img src="https://img.shields.io/badge/status-active-brightgreen?style=flat-square" alt="Status"/>
  <img src="https://img.shields.io/github/last-commit/Abdullah-Racip/offensive-security-walkthroughs?style=flat-square" alt="Last commit"/>
</p>

<h1 align="center">offensive-security-walkthroughs</h1>

<p align="center">
A running log of VulnHub and CTF boxes taken from initial <code>nmap</code> scan to root shell — written up as I go, mostly as coursework for Ethical Hacking at APIIT, partly because I can't stop doing this on weekends too.
</p>

---

### 🧭 How I work a box

<p align="center"><code>recon → enumeration → vulnerability ID → exploitation → privesc → root</code></p>

Every writeup follows that chain end to end — not just the commands that worked, but *why* each step made sense given what recon turned up. No copy-pasted cheat sheets, no skipped reasoning.

### 📂 Structure

Each box gets its own folder:

```
📂 box-name/
 └── README.md   → full writeup: recon → foothold → privesc → root
```

### 🎯 Currently rooted

*(updated as boxes get added)*

| Box | Platform | Difficulty | Status |
|---|---|---|---|
| [DC-1](./dc-1) | VulnHub | Beginner | ✅ Rooted |
| [DC-2](./dc-2) | VulnHub | Beginner | ✅ Rooted |
| [DC-3](./dc-3) | VulnHub | Beginner | ✅ Rooted |
| [Cybersploit: 1](./cybersploit-1) | VulnHub | Easy | ✅ Rooted |
| [Cybersploit: 2](./cybersploit-2) | VulnHub | Easy | ✅ Rooted |

### 🛠️ Stack I lean on

<p align="left">
  <img src="https://img.shields.io/badge/-Nmap-000000?style=flat-square" alt="Nmap"/>
  <img src="https://img.shields.io/badge/-Burp%20Suite-FF6633?style=flat-square" alt="Burp Suite"/>
  <img src="https://img.shields.io/badge/-sqlmap-CC0000?style=flat-square" alt="sqlmap"/>
  <img src="https://img.shields.io/badge/-Metasploit-2596CD?style=flat-square" alt="Metasploit"/>
  <img src="https://img.shields.io/badge/-Wireshark-1679A7?style=flat-square&logo=wireshark&logoColor=white" alt="Wireshark"/>
  <img src="https://img.shields.io/badge/-Kali%20Linux-557C94?style=flat-square&logo=kalilinux&logoColor=white" alt="Kali Linux"/>
</p>

---

> Everything here is done on boxes I have explicit permission to attack —
> VulnHub, CTF platforms, or lab environments. Don't be that person who
> tries this on a system they don't own.

---

<p align="center"><sub>built with too much coffee and Git Bash</sub></p>
