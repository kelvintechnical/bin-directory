# Lab: What Is the `/bin` Directory?

- **Series:** linux-ops-mastery — Linux Filesystem Hierarchy Standard
- **Subjects covered:** Filesystem Hierarchy Standard (FHS), `/bin` purpose for essential user commands, the UsrMerge symlink (`/bin → /usr/bin`), inspecting a directory with `ls -l`, identifying binary types with `file`, listing shared library dependencies with `ldd`, why "essential" binaries live separately from `/usr/bin` historically
- **Career arcs covered:** RHCSA (every shell skill assumes `/bin` is on `$PATH`), RHCE (Ansible's `command:` and `shell:` modules resolve binaries through the same `$PATH`), SRE (rescue-mode and broken-`$PATH` incidents always start at `/bin`), DevOps (container images strip `/bin` aggressively — knowing what each binary does is mandatory), AI/MLOps (Dockerized training images frequently `apk del` or `dnf remove` "unused" binaries and then break in production)
- **Prerequisite:** Basic `ls`, `cd`, `cat`, and a shell on RHEL 9 / Rocky 9 / Ubuntu
- **Time Estimate:** 20 to 35 minutes
- **Difficulty arc:** Task 1 inspect · 2–3 inventory + read · 4–5 demonstrate purpose · 6 capstone audit

---

## Objective

Stop treating `/bin` as a magic word in `$PATH` and start treating it as a **specific directory with a defined purpose in the Filesystem Hierarchy Standard**. By the end of this lab you can list its contents, count its binaries, prove that on modern RHEL 9 it is a symlink to `/usr/bin` (the UsrMerge), classify the file types it contains, and read the shared library dependency chain of any binary inside it.

The goal is not to break anything. `/bin` is read-only by design and this lab is **inspection-only** — no `chmod`, no `rm`, no relocation. You will look, you will count, you will trace, and you will leave with a mental model strong enough to debug a broken `$PATH` from a rescue prompt.

The capstone is the RHCSA-realistic audit prompt: *"If `/bin` were renamed right now, list the first three things that would break, justify each with the binary that would be missing, and show how you would recover from a single-user / rescue shell."*

---

## Concept: Why `/bin` Exists

```
   ┌───────────────────────────────────────────────────────────────┐
   │  FHS root /                                                   │
   ├───────────────────────────────────────────────────────────────┤
   │  /bin    → essential user binaries (ls, cp, cat, bash, ...)   │
   │  /sbin   → essential system binaries (mount, fsck, reboot)    │
   │  /usr/bin → "non-essential" user binaries (the bulk)          │
   │  /usr/sbin → "non-essential" admin binaries                   │
   │                                                               │
   │  On modern RHEL/Fedora (UsrMerge, since ~RHEL 7):             │
   │     /bin   ── symlink ──►  /usr/bin                           │
   │     /sbin  ── symlink ──►  /usr/sbin                          │
   │     /lib   ── symlink ──►  /usr/lib                           │
   │     /lib64 ── symlink ──►  /usr/lib64                         │
   └───────────────────────────────────────────────────────────────┘
```

`/bin` is one of the small set of directories the FHS calls **essential** — it must contain the commands the **system administrator and any user** needs when no other filesystem is mounted. Historically that meant before `/usr` is mounted on a separate disk during early boot or rescue. Modern Linux distributions collapsed this distinction with the **UsrMerge** so `/bin` and `/usr/bin` are the same directory under two names.

> **Why this matters:** When your `$PATH` is wrecked (cron job ran with empty `PATH`, container `ENTRYPOINT` set `PATH=/usr/local/bin` only, rescue shell drops you into `sh` with nothing), the binaries in `/bin` are the ones you must call by absolute path. Knowing which commands live there is the difference between "I can recover this box" and "I need to reboot."

---

## 📜 Why `/bin` Exists — The Story

The split between `/bin` and `/usr/bin` is older than Linux itself. On the original UNIX systems at Bell Labs in the early 1970s, `/bin` held the commands required to bring the system up to a usable single-user state — the shell, basic file utilities, and the editor. As more software was written, the root disk filled up, so the operators at Bell mounted a second disk at `/usr` and moved the "non-essential" binaries there. That hardware accident hardened into a convention: `/bin` for what you need before `/usr` is mounted, `/usr/bin` for the rest.

Linux inherited the convention. The **Filesystem Hierarchy Standard** was first published in **1994** by the Linux-FHS group (later folded into the Linux Foundation) precisely to stop every distribution from inventing its own directory layout. FHS 3.0, the version most current distributions follow, still defines `/bin` as the home of essential command binaries that must be available in single-user mode, and explicitly lists examples like `cat`, `chmod`, `cp`, `date`, `dd`, `echo`, `ln`, `ls`, `mkdir`, `mv`, `ps`, `pwd`, `rm`, `sh`, `stty`, and `uname`.

Around 2012, **Fedora** introduced **UsrMove** (which the wider ecosystem called **UsrMerge**): there was no longer a practical reason to keep two separate root-disk binary directories now that initramfs handles early boot. `/bin`, `/sbin`, `/lib`, and `/lib64` became symbolic links pointing inside `/usr`. RHEL 7 adopted this in 2014, and every modern RHEL, Rocky, Alma, Fedora, and Arch system ships this way. Debian and Ubuntu finished the same migration over the following years.

> **The point of the story:** `/bin` survives today as a **compatibility name**. Scripts written in 1985 that hard-code `/bin/sh` still work. New scripts that use `/usr/bin/env bash` also work. Both paths land in the same physical directory. The directory itself is no longer "the small root-disk subset" — it is just `/usr/bin` under another name.

---

## 👪 The `/bin` Family — Who Lives There

### Essential user binaries you will recognize

| Binary | What it does | Why it must be in `/bin` |
|---|---|---|
| `bash`, `sh` | Bourne / Bourne-Again shell | The first thing init or rescue runs |
| `ls` | List directory contents | You cannot recover what you cannot list |
| `cat` | Concatenate / view files | The minimal file viewer |
| `cp`, `mv`, `rm` | Copy, move, remove files | Filesystem repair primitives |
| `mkdir`, `rmdir` | Make / remove directories | Repair scaffolding |
| `chmod`, `chown` | Change perms / ownership | Recover misconfigured permissions |
| `echo`, `pwd`, `date` | Trivial introspection | Used in nearly every recovery script |
| `ps`, `kill` | Inspect / signal processes | Live triage before logging in fully |
| `mount`, `umount` | Mount filesystems | Required to bring `/usr`, `/var`, `/home` online |
| `grep`, `sed`, `gzip` | Text + compression | Modern FHS allows these in `/bin` for rescue |

### Related directories you will visit

| Directory | Purpose | Relation to `/bin` |
|---|---|---|
| `/usr/bin` | Non-essential user binaries | `/bin` is a symlink to this on UsrMerge systems |
| `/sbin` | Essential admin binaries | Sibling — covered in the sbin-directory lab |
| `/usr/sbin` | Non-essential admin binaries | Symlink target of `/sbin` |
| `/usr/local/bin` | Locally-installed user binaries | Where `make install` lands by default |
| `/lib`, `/lib64` | Shared libraries used by `/bin` | Required for binaries here to actually run |

### Tools that interact with `/bin`

| Tool | What it tells you about `/bin` |
|---|---|
| `ls -l /bin` | Reveals whether `/bin` is a directory or symlink |
| `readlink /bin` | Prints the symlink target if any |
| `file /bin/ls` | Identifies ELF format and architecture |
| `ldd /bin/ls` | Lists the shared libraries the binary loads |
| `which ls`, `command -v ls` | Walks `$PATH` and shows which `ls` wins |
| `rpm -qf /bin/ls` | Identifies the RPM that installed the binary |

> **The point of the family tree:** `/bin` does not live alone. Every binary inside it has dependencies in `/lib64`, an installer package, and a symlink relationship to `/usr/bin`. You will touch all four neighbors during the lab.

---

## 🔬 The Anatomy of `ls -l /bin` — In One Diagram

```
$ ls -l /bin
lrwxrwxrwx. 1 root root 7 Apr 12  2024 /bin -> usr/bin
 │          │ │    │    │  │            │     │
 │          │ │    │    │  │            │     └─ The symlink target: relative path "usr/bin" (i.e. /usr/bin)
 │          │ │    │    │  │            └─ The path being inspected
 │          │ │    │    │  └─ Last modification time of the symlink itself
 │          │ │    │    └─ Size in bytes of the symlink string ("usr/bin" = 7 chars)
 │          │ │    └─ Group owner of the symlink
 │          │ └─ Owner of the symlink (always root on RHEL)
 │          └─ Link count (always 1 for a symlink)
 └─ File type + permissions:
      l → symbolic link
      rwxrwxrwx → mode bits ignored by the kernel for symlinks (target's perms are what matter)
      .         → SELinux context is present (extended attrs)
```

Compare to the inside of the symlink target:

```
$ ls -l /usr/bin | head -n 4
total 312488
-rwxr-xr-x. 1 root root  51648 Mar 22  2024 [
-rwxr-xr-x. 1 root root  35480 Mar 22  2024 alias
-rwxr-xr-x. 1 root root  39960 Mar 22  2024 arch
 │          │ │    │     │     │            │
 │          │ │    │     │     │            └─ Binary name
 │          │ │    │     │     └─ mtime from the installing RPM
 │          │ │    │     └─ Size in bytes (real ELF binary, not a symlink)
 │          │ │    └─ Group
 │          │ └─ Owner
 │          └─ Link count
 └─ `-rwxr-xr-x` = regular file, world-readable + executable, owner-writable
```

> **Reading rule:** On modern RHEL, `ls -l /bin` will almost always show **one symlink line** rather than a directory listing. To see the actual binaries you follow the link with `ls -l /bin/` (trailing slash) or `ls -l /usr/bin/`.

---

## 📚 `/bin` Reference Table

| Task | Command | Notes |
|---|---|---|
| See if `/bin` is a symlink | `ls -ld /bin` | Look for `l` as the first character |
| Print symlink target only | `readlink /bin` | Returns `usr/bin` on RHEL 9 |
| Follow link and list contents | `ls /bin/` (trailing slash) | Or `ls /usr/bin` |
| Count binaries | `ls /usr/bin \| wc -l` | Several hundred on a default RHEL 9 |
| Classify a binary | `file /bin/ls` | ELF 64-bit, dynamically linked |
| Show library deps | `ldd /bin/ls` | Lists `.so` files loaded at runtime |
| Find owning package | `rpm -qf /bin/cp` | `coreutils-...x86_64` |
| Verify package integrity | `rpm -V coreutils` | No output = clean |
| Which `ls` wins in `$PATH` | `command -v ls` | Honors aliases too |
| Walk full `$PATH` for a name | `type -a ls` | Shows all matches in order |

> **Rule one of `/bin`:** Never delete or rename it manually. Even though it is a symlink, removing it on a running system instantly breaks every script that runs `/bin/sh` or `/bin/bash` — including the ones that would help you recover.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | EX200 expects you to know `$PATH`, where standard tools live, and how to find them when `which` is unavailable. |
| **RHCE candidate** | Ansible's `command:` and `shell:` modules use the same `$PATH`. Hard-coded `/bin/...` paths are common in playbooks targeting minimal images. |
| **SRE / Platform** | A wrecked `$PATH` is a top-five "why did cron fail?" answer. Knowing `/bin` contents lets you call binaries by absolute path. |
| **DevOps** | Slim base images (`alpine`, `distroless`, `ubi-minimal`) ship a curated `/bin`. Knowing what's missing is half of dockerfile debugging. |
| **AI / MLOps** | Training containers strip "non-essential" binaries to shrink images, then break when init scripts call `mount` or `gzip`. The `/bin` inventory tells you what is safe to remove. |

---

## 🔧 The 6 Tasks

> Six inspection-only phases that build the **identify → enumerate → classify → trace → audit** habit for `/bin`.

---

### Task 1 — Inspect `/bin` itself

**Purpose:** Confirm whether `/bin` is a real directory or a symlink on your current RHEL 9 system, and capture the metadata you will refer to throughout the lab.

```bash
ls -ld /bin
stat /bin
readlink /bin
file /bin
```

**Human-Readable Breakdown:** Use `ls -ld` to see the symlink line (the `-d` keeps `ls` from descending into the directory), `stat` to read inode metadata, `readlink` to print only the symlink target, and `file` to classify the path itself.

**Reading it left to right:** `ls -ld /bin` returns one line. `stat /bin` prints the inode entry including the symlink target on the last line. `readlink /bin` returns just the target string. `file /bin` returns `symbolic link to usr/bin` on UsrMerge systems or `directory` on older non-UsrMerged systems.

**The story:** Every troubleshooting session starts by establishing ground truth. Is `/bin` a directory? A symlink? Broken? These four commands together rule out three different failure modes in five seconds.

**Expected output:**

```text
lrwxrwxrwx. 1 root root 7 Apr 12  2024 /bin -> usr/bin
  File: /bin -> usr/bin
  Size: 7         	Blocks: 0          IO Block: 4096   symbolic link
Device: fd00h/64768d	Inode: 1572865     Links: 1
Access: (0777/lrwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-05-26 09:14:11.000000000 -0400
Modify: 2024-04-12 03:24:18.000000000 -0400
Change: 2024-04-12 03:24:18.000000000 -0400
 Birth: 2024-04-12 03:24:18.000000000 -0400
usr/bin
/bin: symbolic link to usr/bin
```

**Switches**

| Token | Meaning |
|---|---|
| `ls -ld` | Long format, do not descend into directory |
| `stat` | Print inode metadata (uid, gid, mtime, target) |
| `readlink` | Print symlink target only |
| `file` | Classify file by content (or symlink type) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `/bin` reports as a directory, not symlink | You are on a pre-UsrMerge distro (very old RHEL 6, some embedded distros) — the rest of the lab still works |
| `readlink /bin` is empty | `/bin` is not a symlink on this system — skip Task 3 verification or adjust |
| `Permission denied` on `stat` | Highly unusual — `/bin` is world-readable by FHS; check for SELinux denials |

---

### Task 2 — Inventory the binaries inside `/bin`

**Purpose:** Get a feel for the **size and shape** of `/bin` — how many commands, which ones are most familiar, and how the listing differs between a sparse and a fat install.

```bash
ls /bin/ | head -n 20
ls /bin/ | wc -l
ls /bin/ | grep -E '^(ls|cp|mv|rm|cat|grep|sed|awk|bash|sh)$'
ls -l /bin/ls /bin/cp /bin/cat /bin/bash
```

**Human-Readable Breakdown:** List the first 20 entries to scan the alphabetical head of `/bin`, count the total with `wc -l`, filter for the iconic essentials, then inspect the long-format metadata of four common binaries.

**Reading it left to right:** The trailing slash on `ls /bin/` makes `ls` dereference the symlink and list contents. `wc -l` counts entries. `grep -E '^(...|...)$'` anchors so it matches exact names. `ls -l` for the four named binaries shows real ELF sizes, owners, and mtimes.

**The story:** A default RHEL 9 server install ships several hundred binaries under `/bin`/`/usr/bin`. A minimal container image may have fewer than 40. Knowing the count up front is how you spot "this image is missing something I expected."

**Expected output:**

```text
[
alias
arch
awk
b2sum
base32
base64
basename
basenc
bash
bashbug
bg
bzcat
bzcmp
bzdiff
bzegrep
bzexport
bzfgrep
bzgrep
bzip2
1024
bash
cat
cp
grep
ls
mv
rm
sed
sh
-rwxr-xr-x. 1 root root  144296 Mar 22  2024 /bin/bash
-rwxr-xr-x. 1 root root   51648 Mar 22  2024 /bin/cat
-rwxr-xr-x. 1 root root  153080 Mar 22  2024 /bin/cp
-rwxr-xr-x. 1 root root  147944 Mar 22  2024 /bin/ls
```

**Switches**

| Token | Meaning |
|---|---|
| `ls /bin/` | Follow symlink, list target directory |
| `wc -l` | Count lines (binaries) |
| `grep -E '^(a\|b)$'` | Extended regex, anchored to start and end of line |
| `ls -l file1 file2 ...` | Long listing of multiple files |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ls: cannot access '/bin/'` | Filesystem corruption or `/usr` unmounted — boot rescue media |
| Count is suspiciously low (< 50) | Minimal container image or `ubi-micro` — install `coreutils-single` |
| Binary missing from `grep` filter | Distro packaging changed; try `command -v <name>` instead |

---

### Task 3 — Prove `/bin` is a symlink to `/usr/bin` (UsrMerge)

**Purpose:** Demonstrate that `/bin/ls` and `/usr/bin/ls` are the **same inode** — not two copies — and therefore changes to one are reflected in the other.

```bash
readlink -f /bin
readlink -f /bin/ls
stat -c '%i %n' /bin/ls /usr/bin/ls
ls -l /bin/ls /usr/bin/ls
diff <(ls /bin/ | sort) <(ls /usr/bin/ | sort) | head
```

**Human-Readable Breakdown:** `readlink -f` resolves the full symlink chain. `stat -c '%i %n'` prints inode number plus name — identical inode numbers prove same file. `diff` between the two directory listings should produce no output.

**Reading it left to right:** `readlink -f /bin` walks all symlinks and prints `/usr/bin`. `readlink -f /bin/ls` does the same for the binary, returning `/usr/bin/ls`. `stat -c` formats the output so the inode column lines up. `diff <(...)` uses process substitution to compare the two listings directly.

**The story:** A symlink relationship means there is **one binary on disk**, not two. RPM installs it under `/usr/bin`. The `/bin` symlink makes the historical `/bin/sh` shebang continue to work. Anyone who tries to "harden" a system by deleting `/bin` is in for a very bad day.

**Expected output:**

```text
/usr/bin
/usr/bin/ls
1573420 /bin/ls
1573420 /usr/bin/ls
-rwxr-xr-x. 1 root root 147944 Mar 22  2024 /bin/ls
-rwxr-xr-x. 1 root root 147944 Mar 22  2024 /usr/bin/ls
```

**Switches**

| Token | Meaning |
|---|---|
| `readlink -f` | Canonicalize (follow all symlinks recursively) |
| `stat -c '%i %n'` | Format: inode number, filename |
| `<(...)` | Bash process substitution — treats command output as a file |
| `diff` | Line-by-line comparison |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Inode numbers differ | You are on a non-UsrMerged distro — note it and continue |
| `diff` shows many lines | Something has unsymlinked `/bin` (extremely rare) — investigate before changing anything |
| `readlink -f` returns empty | Broken symlink — re-create with `ln -s usr/bin /bin` only from rescue media |

---

### Task 4 — Classify file types under `/bin` with `file`

**Purpose:** Confirm that everything in `/bin` is either an **ELF 64-bit binary**, a **shell script**, or a **symlink** — and learn to read `file` output well enough to spot a non-binary impostor.

```bash
file /bin/ls
file /bin/bash
file /bin/sh
file /bin/awk
file /bin/zcat
file /bin/* 2>/dev/null | awk -F: '{print $2}' | sort | uniq -c | sort -rn | head
```

**Human-Readable Breakdown:** Run `file` on five well-known commands first to learn the canonical outputs, then `file /bin/*` to classify everything, count each unique classification, and surface the most common types.

**Reading it left to right:** `file` reads the magic bytes (the first few bytes of the file) to determine type. `awk -F: '{print $2}'` strips the filename prefix. `sort | uniq -c` groups identical type strings and counts them. `sort -rn` ranks by count descending.

**The story:** Most `/bin` entries on RHEL 9 are `ELF 64-bit LSB pie executable, x86-64`. A handful are POSIX shell scripts (`zcat`, `bzgrep`). A couple are symlinks (`sh → bash`, `awk → gawk`). If you spot anything else — say, a `Python script` or `Bourne-Again shell script` in `/bin` — that is a finding to investigate.

**Expected output:**

```text
/bin/ls: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=..., for GNU/Linux 3.2.0, stripped
/bin/bash: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=..., for GNU/Linux 3.2.0, stripped
/bin/sh: symbolic link to bash
/bin/awk: symbolic link to gawk
/bin/zcat: POSIX shell script, ASCII text executable
    480  ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=..., for GNU/Linux 3.2.0, stripped
     61  POSIX shell script, ASCII text executable
     27  symbolic link to ...
      8  Perl script text executable
```

**Switches**

| Token | Meaning |
|---|---|
| `file PATH` | Identify by magic bytes |
| `awk -F:` | Set field separator to colon |
| `uniq -c` | Count consecutive duplicate lines |
| `sort -rn` | Reverse, numeric sort |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `file` outputs only `data` | Binary is corrupt; reinstall the owning RPM with `dnf reinstall` |
| `2>/dev/null` lines on screen | You omitted it — silence with `2>/dev/null` to discard noise |
| `awk` shows weird counts | Two binaries with the same type but different build IDs — that is expected |

---

### Task 5 — Trace a binary's library dependencies with `ldd`

**Purpose:** See exactly which shared libraries `/bin/ls` loads at runtime. This is the bridge between this lab and the upcoming `/lib64` lab.

```bash
ldd /bin/ls
ldd /bin/bash
ldd /bin/cat | wc -l
ldd /bin/cp | awk '{print $1}' | sort -u
ldd /bin/ls | grep -E 'libc|ld-linux' | head
```

**Human-Readable Breakdown:** Run `ldd` against several binaries to see the load list, count how many libraries one of them needs, deduplicate the library names, and finally filter for the two most critical ones — `libc.so.6` (C standard library) and `ld-linux-x86-64.so.2` (dynamic linker).

**Reading it left to right:** `ldd` is itself a shell script that sets `LD_TRACE_LOADED_OBJECTS=1` and runs the binary — that environment variable tells the dynamic linker to print the resolution map instead of executing. The output shows each `libname.so => /resolved/path (0xaddress)`.

**The story:** Every binary in `/bin` depends on `/lib64/libc.so.6` and `/lib64/ld-linux-x86-64.so.2`. Delete or break either of those files and **every dynamically linked binary on the system stops working** — including the commands you would use to recover. This is why `/bin` and `/lib64` are co-required in FHS.

**Expected output:**

```text
	linux-vdso.so.1 (0x00007ffe...)
	libselinux.so.1 => /lib64/libselinux.so.1 (0x00007f...)
	libcap.so.2 => /lib64/libcap.so.2 (0x00007f...)
	libc.so.6 => /lib64/libc.so.6 (0x00007f...)
	libpcre2-8.so.0 => /lib64/libpcre2-8.so.0 (0x00007f...)
	/lib64/ld-linux-x86-64.so.2 (0x00007f...)
	linux-vdso.so.1 (0x00007ffd...)
	libtinfo.so.6 => /lib64/libtinfo.so.6 (0x00007f...)
	libc.so.6 => /lib64/libc.so.6 (0x00007f...)
	/lib64/ld-linux-x86-64.so.2 (0x00007f...)
4
	/lib64/ld-linux-x86-64.so.2
	libacl.so.1
	libattr.so.1
	libc.so.6
	libselinux.so.1
	linux-vdso.so.1
	libc.so.6 => /lib64/libc.so.6 (0x00007fd9a4e00000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fd9a5024000)
```

**Switches**

| Token | Meaning |
|---|---|
| `ldd PATH` | Print shared library dependencies |
| `awk '{print $1}'` | First field (the SONAME) |
| `sort -u` | Sort + unique in one step |
| `grep -E` | Extended regex (multiple alternatives) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `not a dynamic executable` | Binary is statically linked (e.g. `/bin/busybox`) — that's fine |
| `not found` on a library | Broken install — reinstall with `dnf reinstall <pkg>` |
| `ldd` runs the binary (security warning) | True — never `ldd` an untrusted binary; use `objdump -p` instead |

---

### Task 6 — Capstone: `/bin` recovery audit

**Task statement:** *"If `/bin` were renamed right now, list the first three things that would break, justify each with the binary that would be missing, and show how you would recover from a single-user / rescue shell."*

**Purpose:** Tie inspection (Tasks 1–5) into operational reasoning. Predict failures, justify with evidence, sketch the recovery path.

```bash
echo "--- 1. SHELLS THAT BREAK ---"
ls -l /bin/sh /bin/bash 2>&1
echo
echo "--- 2. CORE FILE TOOLS THAT BREAK ---"
for cmd in ls cp mv rm cat; do printf "%-6s -> " "$cmd"; readlink -f "/bin/$cmd"; done
echo
echo "--- 3. MOUNT + RECOVERY TOOLS THAT BREAK ---"
for cmd in mount umount sed grep gzip; do printf "%-8s -> " "$cmd"; readlink -f "/bin/$cmd"; done
echo
echo "--- 4. RECOVERY PATH (read-only audit) ---"
echo "Rescue shell would run: /usr/bin/ln -s usr/bin /bin"
echo "Verify after fix: ls -l /bin && /bin/ls --version | head -1"
```

**Human-Readable Breakdown:** Build a four-section audit report. Section 1 lists the shells that would vanish (every login + every `#!/bin/sh` script). Section 2 lists the file primitives. Section 3 lists the recovery primitives that would themselves be missing. Section 4 records the recovery one-liner you would type from rescue media.

**Reading it left to right:** `readlink -f` for each binary resolves to `/usr/bin/...` — proving the canonical location is preserved in `/usr/bin`. The recovery one-liner `ln -s usr/bin /bin` recreates the symlink with a **relative** target, which matches the way RHEL installs it.

**The story:** This is the audit you produce on the back of a napkin during a 3 a.m. incident. The point is not to break `/bin` — the point is to know **exactly** which absolute paths in `/usr/bin` you would call directly to put it back. The recovery never needs `/bin/ls`; it always uses `/usr/bin/ln`.

**Expected output:**

```text
--- 1. SHELLS THAT BREAK ---
lrwxrwxrwx. 1 root root 4 Apr 12  2024 /bin/sh -> bash
-rwxr-xr-x. 1 root root 144296 Mar 22  2024 /bin/bash

--- 2. CORE FILE TOOLS THAT BREAK ---
ls     -> /usr/bin/ls
cp     -> /usr/bin/cp
mv     -> /usr/bin/mv
rm     -> /usr/bin/rm
cat    -> /usr/bin/cat

--- 3. MOUNT + RECOVERY TOOLS THAT BREAK ---
mount    -> /usr/bin/mount
umount   -> /usr/bin/umount
sed      -> /usr/bin/sed
grep     -> /usr/bin/grep
gzip     -> /usr/bin/gzip

--- 4. RECOVERY PATH (read-only audit) ---
Rescue shell would run: /usr/bin/ln -s usr/bin /bin
Verify after fix: ls -l /bin && /bin/ls --version | head -1
```

**Cleanup**

```bash
# nothing destructive — this lab was inspection-only
# no /tmp files written, no symlinks created, no packages installed
echo "lab complete — /bin untouched"
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `readlink -f` returns empty for a binary | That command does not exist in your install — `dnf install coreutils` |
| Audit reports wrong canonical paths | You are on a non-UsrMerged system — substitute `/bin/...` in the recovery one-liner |
| Recovery `ln` syntax confusion | Always use `ln -s <relative-target> /bin` — never an absolute `/usr/bin` target inside a chroot |

---

## 🔍 `/bin` Decision Guide

```
Looking for a binary on a RHEL 9 system?
  │
  ├── "Is it an essential user tool? (ls, cp, cat, bash)"
  │       └── ✅ /bin/<name>           (resolves to /usr/bin/<name>)
  │
  ├── "Is it an admin tool? (fdisk, reboot, iptables)"
  │       └── ✅ /sbin/<name>          (resolves to /usr/sbin/<name>)
  │
  ├── "Is it third-party / locally-installed?"
  │       └── ✅ /usr/local/bin/<name>
  │
  ├── "I don't know — let the shell find it"
  │       └── ✅ command -v <name>      (honors aliases + functions)
  │       └── ✅ type -a <name>         (shows every match in $PATH)
  │
  └── "I need the absolute path for a script"
          └── ✅ readlink -f $(command -v <name>)
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Inspect `/bin` with `ls -ld`, `stat`, `readlink`, and `file`
- [ ] 02 Inventory `/bin` contents and count binaries with `wc -l`
- [ ] 03 Prove `/bin` is a symlink to `/usr/bin` via matching inodes
- [ ] 04 Classify `/bin` contents with `file` and rank types by count
- [ ] 05 Trace `/bin/ls` shared library dependencies with `ldd`
- [ ] 06 Produce the `/bin` recovery audit report and recovery one-liner

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Treating `/bin` as a real directory | Confused when `ls -l /bin` shows one line | Use `ls -ld /bin` and `readlink /bin` first |
| Forgetting the trailing slash | `ls /bin` shows symlink, not contents | Use `ls /bin/` or `ls /usr/bin` |
| Deleting `/bin` "to clean up" | Every login breaks instantly | Never modify FHS roots manually |
| Confusing `/bin` and `/sbin` | Looking for `fdisk` in `/bin` | Admin binaries live in `/sbin` |
| Assuming `ldd` is safe on untrusted ELF | `ldd` can execute attacker code | Use `objdump -p` or `readelf -d` instead |
| Hardcoding `/bin/python` in scripts | Modern distros only ship `/usr/bin/python3` | Use `#!/usr/bin/env python3` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Know that `/bin`, `/sbin`, `/lib`, `/lib64` are symlinks into `/usr` on modern RHEL. Be ready to explain UsrMerge in one sentence.

**RHCE candidate**
- When writing Ansible roles for minimal images, never assume `/bin/python` exists. Use `ansible_python_interpreter: /usr/bin/python3` for RHEL 9 targets.

**SRE / Platform interview**
- "Cron job ran with empty `PATH` and failed" — walk through the audit: list binaries cron called, resolve each to `/usr/bin/...`, hard-code the absolute paths or set `PATH=/usr/bin:/usr/sbin` in the crontab.

**DevOps**
- Distroless and `ubi-micro` images strip `/bin`. Know which tools (`sh`, `ls`, `cat`) you can no longer assume.

**AI / MLOps**
- Training containers built on `nvidia/cuda` keep `/bin` but strip `man`, `info`, and locale files. The `file /bin/*` audit reveals what is still present before you push the image.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| [/sbin](https://github.com/kelvintechnical/sbin-directory) | Sibling — admin binaries that resolve into `/usr/sbin` |
| [/lib](https://github.com/kelvintechnical/lib-directory) | Where the libraries `/bin/*` load from live |
| [/lib64](https://github.com/kelvintechnical/lib64-directory) | 64-bit shared libraries — every `/bin` binary links here |
| [/usr](https://github.com/kelvintechnical/usr-directory) | The canonical home of the binaries `/bin` symlinks to |
| [/etc](https://github.com/kelvintechnical/etc-directory) | Configuration files those binaries read |
| [/boot](https://github.com/kelvintechnical/boot-directory) | Kernel images that init these binaries |
| [/home](https://github.com/kelvintechnical/home-directory) | User home directories — where `$PATH` is set per user |
| [/root](https://github.com/kelvintechnical/root-directory) | Root home — first place to land in rescue mode |
| [/var](https://github.com/kelvintechnical/var-directory) | Logs that record `/bin/*` invocations via auditd |
| [/tmp](https://github.com/kelvintechnical/tmp-directory) | Scratch space where many `/bin` commands write |
| [/opt](https://github.com/kelvintechnical/opt-directory) | Add-on application binaries outside FHS essentials |
| [/srv](https://github.com/kelvintechnical/srv-directory) | Service data directories |
| [/dev](https://github.com/kelvintechnical/dev-directory) | Device nodes the binaries open |
| [/proc](https://github.com/kelvintechnical/proc-directory) | Kernel introspection — `ps` reads from here |
| [/sys](https://github.com/kelvintechnical/sys-directory) | Kernel object exposure |
| [/run](https://github.com/kelvintechnical/run-directory) | Runtime state (PID files, sockets) |
| [/media](https://github.com/kelvintechnical/media-directory) | Auto-mounted removable media |
| [/mnt](https://github.com/kelvintechnical/mnt-directory) | Manual mount points |
| [/afs](https://github.com/kelvintechnical/afs-directory) | AFS distributed filesystem mount point |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
