# Investigating Windows — Study Notes

TryHackMe room. Windows Server 2016 host compromised on 03/02/2019. Live triage
of a running box over RDP, no disk image, no memory capture.

---

## Answers and where they came from

| # | Question | Answer | Source |
|---|---|---|---|
| 1 | Version and year | Windows Server 2016 | Settings → System → About (Edition field) |
| 2 | Last user logged in | Administrator | `net user <name>` → Last logon, compared across accounts |
| 3 | John's last logon | 03/02/2019 5:48:32 PM | `net user John` |
| 4 | IP contacted at boot | 10.34.2.3 | `HKLM\...\CurrentVersion\Run` → `UpdateSvc` value |
| 5 | Two admin accounts | Guest, Jenny | `net localgroup Administrators` |
| 6 | Malicious scheduled task | Clean file system | `schtasks /query` → root-level task running `nc.ps1` |
| 7 | File the task ran daily | nc.ps1 | Task XML `<Command>` |
| 8 | Jenny's last logon | Never | `net user Jenny` |
| 9 | Compromise date | 03/02/2019 | Task `<RegistrationInfo><Date>`, corroborated by `C:\TMP` timestamps |
| 10 | First 4672 event | 03/02/2019 4:04:49 PM | Security log, Event ID 4672 (log had rolled on our instance) |
| 11 | Password dumping tool | mimikatz | `mim.exe`, `mim-out.txt`, LSASS dump in `C:\TMP` |
| 12 | C2 server IP | 76.32.97.132 | hosts file |
| 13 | Web shell extension | .jsp | IIS web root |
| 14 | Last port opened | 1337 | `wf.msc` → inbound rule with no Group |
| 15 | DNS poisoning target | google.com | hosts file |

---

## Commands worth memorizing

### System identification
```
systeminfo                  # OS name, version, install date, hotfixes, domain
winver                      # quick GUI version popup
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
```

### Accounts
```
net user                    # list all local accounts
net user <name>             # detail: last logon, created, group membership
net user <name> | findstr /i "logon"
net localgroup Administrators
```
`Last logon: Never` on a privileged account is a strong indicator of attacker
provisioning — a real person's account gets used.

### Scheduled tasks
```
schtasks /query /fo LIST | findstr /i "TaskName"
schtasks /query /fo LIST | findstr /i "TaskName" | findstr /v /i "Microsoft"
schtasks /query /tn "<name>" /fo LIST /v
schtasks /query /tn "<name>" /xml
schtasks /query /tn "<name>" /xml | findstr /i "Command Arguments Trigger"
taskschd.msc                # GUI
```

### Autostart / persistence locations
```
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /s
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce" /s
reg query "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /s
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
dir /s "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
dir /s C:\Windows\System32\GroupPolicy\Machine\Scripts
```

### Network config
```
type C:\Windows\System32\drivers\etc\hosts
netsh advfirewall firewall show rule name=all dir=in
wf.msc                      # GUI, far easier for scanning rules
netstat -anob               # active listeners + owning process
```

### Filesystem
```
dir /a C:\TMP
dir /s /b C:\*.evtx
findstr /s /i "searchterm" C:\path\*
```

---

## Event IDs

| ID | Meaning |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4672 | Special privileges assigned to new logon (admin-level session) |
| 4688 | New process created |
| 4720 | User account created |
| 4726 | User account deleted |
| 4732 | Member added to security-enabled local group |
| 1102 | Audit log cleared |

Event Viewer: `eventvwr.msc` → Windows Logs → Security → Filter Current Log →
enter the ID. Sort by Date and Time to get chronological order.

CLI equivalent:
```
wevtutil qe Security /q:"*[System[(EventID=4672)]]" /f:text /c:200
```

---

## Attacker toolkit found in C:\TMP

Every file timestamped 03/02/2019, which is what pinned the compromise date.

| File | Purpose |
|---|---|
| `mim.exe` | mimikatz, renamed |
| `mim-out.txt` | mimikatz output — harvested credentials |
| `somethingwindows.dmp` | 40 MB LSASS memory dump, mimikatz input |
| `nc.ps1` | netcat, run daily via scheduled task, listening on 1348 |
| `p.exe` | PsExec-style remote execution, called from the Run key |
| `xCmd.exe` | another remote execution utility |
| `nbtscan.exe` | NetBIOS network discovery |
| `WMIBackdoor.ps1` | WMI-based persistence |
| `schtasks-backdoor.ps1` | scheduled-task persistence |
| `d.txt`, `sys.txt`, `scan*.tmp` | recon output |

`C:\TMP` itself is the first red flag. Windows has no such directory by default.
Same class of indicator: `C:\Users\Public`, `C:\Windows\Temp`, `%APPDATA%`.

---

## How to judge whether something is malicious

Applied to scheduled tasks here, but the framework generalizes to services,
registry entries, and running processes.

| Signal | Question to ask |
|---|---|
| Path | Real system directory, or somewhere world-writable? |
| Binary identity | Signed Microsoft file, or dropped there? |
| Arguments | Network flags, an IP, `-enc`, `-WindowStyle Hidden`, `-ExecutionPolicy Bypass`? |
| Name vs behavior | Does what it does match what it's called? |
| Trigger | Boot / logon / high frequency = persistence. On-demand = probably dormant. |
| Run as | SYSTEM or Administrator for something trivial? |
| Location in hierarchy | Legit Windows tasks live under `\Microsoft\Windows\`. Root-level ones were created by a person. |

**Worked example — "Clean file system" vs "update windows"**

`Clean file system` runs `C:\TMP\nc.ps1 -l 1348`. Fails on three counts at once:
non-standard path, netcat listening on a port, and a name that has nothing to do
with what it does. The description ("A task to clean old files of the system")
is cover text.

`update windows` runs `C:\Program Files (x86)\Internet Explorer\ieinstal.exe`.
Legitimate signed binary, real path, no arguments, on-demand trigger, never
executed. Looks odd — why would anyone create this? — but nothing proves malice.
Correct call: flag as suspicious, don't conclude. Not every anomaly is an
indicator.

---

## The hosts file finding

```
10.2.2.2       update.microsoft.com
127.0.0.1      www.virustotal.com
127.0.0.1      www.www.com
127.0.0.1      dci.sophosupd.com
76.32.97.132   google.com
76.32.97.132   www.google.com
```

Three distinct objectives in one file:

1. **Redirect** — google.com points at the attacker's C2. Any browse to Google
   from this box hits their server instead.
2. **Blind the defenses** — virustotal.com and the Sophos update endpoint
   blackholed to localhost. No sample submission, no AV signature updates.
3. **Hijack updates** — Windows Update repointed to an internal host.

Name resolution order on Windows is: DNS cache → **hosts file** → DNS server.
An entry in hosts short-circuits DNS entirely, so no network-level attack is
required. Writing to the file needs admin, which they already had. This is a
post-compromise action, not the initial access vector.

Terminology note: strictly, "DNS cache poisoning" means corrupting a resolver's
cache so it serves bad answers to many clients. What happened here is local host-file
hijacking — single machine, no protocol manipulation. The room uses the loose sense.
Worth keeping the distinction straight.

A clean hosts file is entirely `#` comments. Any active line deserves scrutiny.
File integrity monitoring on that path is standard practice for this reason.

---

## Firewall rule finding

Inbound rule: **"Allow outside connections for development"**, TCP, local port
**1337**, no Group assigned.

The missing Group value is the tell. Every built-in Windows rule belongs to a
group (Core Networking, Network Discovery, Cast to Device, etc.). Manually
created rules have that column blank, which makes them easy to spot when you
sort by it.

Note there were two listeners: 1348 (nc.ps1, via the daily task) and 1337 (with
the firewall exception). Only 1337 was reachable externally.

---

## Attack chain reconstructed

1. Initial access via a web shell uploaded to the IIS site
2. Credential harvesting — LSASS dumped, mimikatz run against it
3. Privilege escalation — Jenny created and added to Administrators
4. Persistence, layered:
   - Registry Run key (`UpdateSvc` → `p.exe` against 10.34.2.3)
   - Scheduled task (`Clean file system` → `nc.ps1 -l 1348`)
   - WMI backdoor script staged
5. Defense evasion — hosts file entries killing AV updates and VirusTotal
6. C2 and lateral movement — 76.32.97.132 external, 10.34.2.3 internal,
   `nbtscan` for discovery, `xCmd`/`p.exe` for remote execution
7. Firewall exception opened on 1337 for inbound access

Note the redundancy. Multiple independent persistence mechanisms means removing
any one of them doesn't evict the attacker. Real incident response has to
enumerate all of them before remediating, or the box gets reinfected.

---

## Process lessons

**Read what's already on screen.** The `nc.ps1` command was visible in XML output
two screens before we identified it. Scrolling back beat re-running commands.

**Check answer-format hints.** Asterisk counts narrowed candidates repeatedly —
`*****, *****` ruled out John (4 chars) for the admin question; `*****` for
Jenny's logon pointed at "Never" before the command was even run.

**Filter output, don't page through it.** `findstr` on `schtasks` and `netsh`
turned hundreds of lines into a handful. Field names matter: it's `Task To Run`
with spaces in `/fo LIST` output, `<Command>` in XML.

**Live logs roll.** The Security log had 93,000 recent events and no 4672 entries
from 2019. On a real engagement you'd capture the .evtx immediately rather than
relying on the live log — evidence expires.

**GUI beats CLI for scanning, CLI beats GUI for filtering.** `wf.msc` made the
odd firewall rule obvious in seconds; `netsh` output would have taken far longer
to skim. The reverse held for scheduled tasks.

---

## Follow-up worth doing

- Read `mim-out.txt` to see what credentials were actually exposed
- Read `WMIBackdoor.ps1` and `schtasks-backdoor.ps1` to understand those techniques
- Look up the MITRE ATT&CK IDs for each stage above (T1003 credential dumping,
  T1053 scheduled task, T1547 Run keys, T1562 impair defenses)
- Try the same triage on a Linux box to contrast the artifact locations
