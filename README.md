# What is the /bin Directory?

**Linux Filesystem Hierarchy Standard (FHS)** | Lab 01  
**Part of:** [Linux-Filesystem-Hierarchy-Standard](https://github.com/kelvintechnical/Linux-Filesystem-Hierarchy-Standard-)  
**Time Estimate:** 10–15 minutes

---

## 🎯 Objective

Understand what the `/bin` directory is, what lives inside it, why it exists, and how it differs from `/usr/bin` — through hands-on exploration.

---

## 🧠 Big Idea — What is /bin?

`/bin` stands for **Binaries**. It contains the essential command-line programs that every user on the system needs to run — even during system recovery when nothing else is available.

> Think of `/bin` as the **bare minimum toolkit** — the commands the system cannot live without.

**Key rule:** If a command needs to work **before** the full filesystem is mounted, it lives in `/bin`.

---

## 📚 Command Decision Map

| Task | Command |
|---|---|
| List contents of /bin | `ls /bin` |
| Count how many binaries are in /bin | `ls /bin \| wc -l` |
| Find where a command lives | `which <command>` |
| See full path of a command | `type <command>` |
| Check if /bin is a symlink | `ls -la / \| grep bin` |
| Read what a command does | `man <command>` |

---

## 🔧 Steps

### Step 1 — List the contents of /bin

```bash
ls /bin
```

**What this does:**
- `ls` — list directory contents
- `/bin` — the target directory

> You will see hundreds of familiar commands: `ls`, `cp`, `mv`, `cat`, `echo`, `grep`, `bash`, and more.

---

### Step 2 — Count how many binaries are in /bin

```bash
ls /bin | wc -l
```

**Command explained:**

| Part | Meaning |
|---|---|
| `ls /bin` | List all files in /bin |
| `\|` | Pipe — send that output to the next command |
| `wc -l` | Count the number of lines (one per file) |

---

### Step 3 — Find where a familiar command lives

```bash
which ls
which cp
which grep
```

**What this does:**
- `which` — searches your PATH and returns the full path of a command
- Confirms these commands live in `/bin`

---

### Step 4 — Check if /bin is a symlink on modern RHEL

```bash
ls -la / | grep bin
```

**What this does:**

| Part | Meaning |
|---|---|
| `ls -la /` | List root directory with hidden files and details |
| `\|` | Pipe output forward |
| `grep bin` | Filter only lines containing "bin" |

**Expected output on RHEL 9:**
```
lrwxrwxrwx.  1 root root    7 ... bin -> usr/bin
lrwxrwxrwx.  1 root root    8 ... sbin -> usr/sbin
```

**Output explained:**

| Detail | Meaning |
|---|---|
| `lrwxrwxrwx` | `l` = this is a symbolic link (symlink) |
| `bin -> usr/bin` | `/bin` is just a pointer to `/usr/bin` |
| `sbin -> usr/sbin` | `/sbin` is just a pointer to `/usr/sbin` |

> On modern RHEL 9, `/bin` and `/usr/bin` are the **same directory** — `/bin` is a symlink to `/usr/bin`. This is called **UsrMerge** and was introduced to simplify the filesystem.

---

### Step 5 — Explore a specific binary

```bash
which cat
file /bin/cat
man cat
```

**Command explained:**

| Command | Meaning |
|---|---|
| `which cat` | Shows the full path of the `cat` command |
| `file /bin/cat` | Shows what type of file it is (ELF binary = compiled executable) |
| `man cat` | Opens the manual page — press `q` to quit |

---

### Step 6 — Run a few /bin commands directly by full path

```bash
/bin/ls /etc
/bin/echo "Hello from /bin"
/bin/cat /etc/redhat-release
```

**What this proves:**
- You don't need to type just `ls` — the shell finds it via PATH
- Calling by full path (`/bin/ls`) bypasses PATH entirely and runs the binary directly
- This is how the system runs commands during early boot before PATH is configured

---

## ✅ Lab Checklist

- [ ] `ls /bin` shows essential system commands
- [ ] `ls /bin | wc -l` returns a count of binaries
- [ ] `which ls` returns `/bin/ls` or `/usr/bin/ls`
- [ ] `ls -la / | grep bin` shows `/bin -> usr/bin` symlink
- [ ] `file /bin/cat` confirms it is an ELF executable
- [ ] Commands run successfully using full `/bin/` path

---

## 🧠 /bin vs /usr/bin — What's the Difference?

| Directory | Purpose | When Available |
|---|---|---|
| `/bin` | Essential commands for all users | Available at boot, before full filesystem mount |
| `/usr/bin` | Non-essential user commands and apps | Available after full system startup |
| On RHEL 9 | `/bin` is a symlink to `/usr/bin` | They are the same — UsrMerge |

> On older Linux systems these were separate. On modern RHEL 9 they are unified via symlink.

---

## ⚠️ Common Pitfalls

| Mistake | Fix |
|---|---|
| Expecting `/bin` and `/usr/bin` to be different on RHEL 9 | They are the same — `/bin` is a symlink |
| Confusing `/bin` with `/sbin` | `/bin` = all users; `/sbin` = admin/root only |
| Deleting files from `/bin` | Never — breaks the entire system |

---

## 📌 Exam Tips

- Know that on RHEL 9, `/bin → usr/bin` — this comes up in troubleshooting questions.
- `which <command>` is the fastest way to find where a command lives.
- If a command isn't found, check if `/bin` or `/usr/bin` is in your `$PATH` with `echo $PATH`.

---

## 🔗 Series Index

| # | Directory | Repo |
|---|---|---|
| 👉 01 | `/bin` | **You are here** |
| 02 | `/sbin` | [what-is-the-sbin-directory](https://github.com/kelvintechnical/what-is-the-sbin-directory) |
| 03 | `/lib` | [what-is-the-lib-directory](https://github.com/kelvintechnical/what-is-the-lib-directory) |
| 04 | `/lib64` | [what-is-the-lib64-directory](https://github.com/kelvintechnical/what-is-the-lib64-directory) |
| 05 | `/usr` | [what-is-the-usr-directory](https://github.com/kelvintechnical/what-is-the-usr-directory) |
| 06 | `/etc` | [what-is-the-etc-directory](https://github.com/kelvintechnical/what-is-the-etc-directory) |
| 07 | `/boot` | [what-is-the-boot-directory](https://github.com/kelvintechnical/what-is-the-boot-directory) |
| 08 | `/home` | [what-is-the-home-directory](https://github.com/kelvintechnical/what-is-the-home-directory) |
| 09 | `/root` | [what-is-the-root-directory](https://github.com/kelvintechnical/what-is-the-root-directory) |
| 10 | `/var` | [what-is-the-var-directory](https://github.com/kelvintechnical/what-is-the-var-directory) |
| 11 | `/tmp` | [what-is-the-tmp-directory](https://github.com/kelvintechnical/what-is-the-tmp-directory) |
| 12 | `/opt` | [what-is-the-opt-directory](https://github.com/kelvintechnical/what-is-the-opt-directory) |
| 13 | `/srv` | [what-is-the-srv-directory](https://github.com/kelvintechnical/what-is-the-srv-directory) |
| 14 | `/dev` | [what-is-the-dev-directory](https://github.com/kelvintechnical/what-is-the-dev-directory) |
| 15 | `/proc` | [what-is-the-proc-directory](https://github.com/kelvintechnical/what-is-the-proc-directory) |
| 16 | `/sys` | [what-is-the-sys-directory](https://github.com/kelvintechnical/what-is-the-sys-directory) |
| 17 | `/run` | [what-is-the-run-directory](https://github.com/kelvintechnical/what-is-the-run-directory) |
| 18 | `/media` | [what-is-the-media-directory](https://github.com/kelvintechnical/what-is-the-media-directory) |
| 19 | `/mnt` | [what-is-the-mnt-directory](https://github.com/kelvintechnical/what-is-the-mnt-directory) |
| 20 | `/afs` | [what-is-the-afs-directory](https://github.com/kelvintechnical/what-is-the-afs-directory) |

---

## 🔗 Part of Linux Ops Mastery

- [Linux-Filesystem-Hierarchy-Standard](https://github.com/kelvintechnical/Linux-Filesystem-Hierarchy-Standard-)
- [Linux Ops Mastery](https://github.com/kelvintechnical/linux-ops-mastery)

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
