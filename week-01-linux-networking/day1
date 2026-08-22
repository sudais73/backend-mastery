# Week 1, Day 1: Linux Fundamentals & Unix Pipes

## Overview
**Goal:** Understand how Linux works at its core — the filesystem, processes, permissions, and the Unix philosophy of chaining small tools together.

**What you'll learn:** The foundation that 90% of backend problems sit on top of.

---

## Part 1: Linux Filesystem Hierarchy

### Mental Model
Linux has **ONE filesystem hierarchy** starting at `/` (root). Unlike Windows (C:\, D:\, E:\), everything branches from a single root tree.

### Key Directories

| Directory | Purpose | Example |
|-----------|---------|---------|
| `/` | Root of entire filesystem | Everything starts here |
| `/home` | Where regular users live | Your files go in `/home/umer_muhammed` |
| `/mnt` | Mount points for external filesystems | Windows C:\ drive is at `/mnt/c` |
| `/bin` | Essential programs/executables | `ls`, `grep`, `cat` live here |
| `/etc` | System configuration files | Settings for the OS |
| `/root` | Admin user's home (not regular users!) | System admin's private space |

### Exploring the Filesystem

**Command:** `ls /`
```
bin  boot  dev  etc  home  init  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  snap  srv  sys  tmp  usr  var
```

**What this tells you:** Top-level directories of your system.

---

**Command:** `ls /home`
```
umer_muhammed
```

**What this tells you:** Regular users have their homes in `/home/`. You are `umer_muhammed`.

---

**Command:** `ls /mnt`
```
c  wsl  wslg
```

**What this tells you:** Your Windows C:\ drive is mounted at `/mnt/c`. You can access it from Linux!

---

**Command:** `ls /bin | head -20`
```
NF
X11
[
aa-enabled
aa-exec
aa-features-abi
add-apt-repository
addr2line
apport-bug
apport-cli
apport-collect
apport-unpack
appstreamcli
apropos
apt
apt-add-repository
apt-cache
apt-cdrom
apt-config
apt-get
```

**What this tells you:** Programs/executables that your system uses. When you type `apt`, Linux finds `/bin/apt` and runs it.

---

## Part 2: File Ownership & Permissions

### Mental Model
Every file in Linux has three labels:
1. **Who owns it?** (user)
2. **What group owns it?** (group)
3. **Who can do what?** (permissions)

### The Permission String

When you run `ls -la`, you see something like:

```bash
ls -la ~
```

Output:
```
total 28
drwxr-x--- 4 umer_muhammed umer_muhammed 4096 Aug 22 13:37 .
drwxr-xr-x 3 root          root          4096 Aug 22 13:36 ..
-rw-r--r-- 1 umer_muhammed umer_muhammed  220 Aug 22 13:36 .bash_logout
-rw-r--r-- 1 umer_muhammed umer_muhammed 3771 Aug 22 13:36 .bashrc
drwxr-x--- 4 umer_muhammed umer_muhammed 4096 Aug 22 13:37 .cache
drwxr-x--- 3 umer_muhammed umer_muhammed 4096 Aug 22 13:37 .config
-rw-rw-r-- 1 umer_muhammed umer_muhammed    0 Aug 22 13:37 .motd_shown
-rw-r--r-- 1 umer_muhammed umer_muhammed  807 Aug 22 13:36 .profile
```

### Breaking Down One Line

```
-rw-r--r-- 1 umer_muhammed umer_muhammed  220 Aug 22 13:36 .bash_logout
```

| Part | Meaning |
|------|---------|
| `-` | Regular file (not a directory) |
| `rw-` | **Owner (you):** read & write, NO execute |
| `r--` | **Group:** read only |
| `r--` | **Others:** read only |
| `1` | Hard link count (ignore) |
| `umer_muhammed` | Owner (who owns this file) |
| `umer_muhammed` | Group (which group owns it) |
| `220` | File size in bytes |
| `Aug 22 13:36` | When it was last modified |
| `.bash_logout` | Filename |

### Permission Symbols

- **`r`** (read) = can view contents
- **`w`** (write) = can modify/delete
- **`x`** (execute) = can run (for files) or enter (for directories)
- **`-`** = no permission

### The 3-Character Pattern

```
-rw-r--r--
 │  │  │
 │  │  └─ Others (everyone else) — r--
 │  └───── Group — r--
 └──────── Owner (you) — rw-
```

### Real-World Example

```
drwxr-x--- 4 umer_muhammed umer_muhammed 4096 Aug 22 13:37 .
```

- **You:** Full access (`rwx` = read, write, execute/enter)
- **Group:** Can only read and enter (`r-x`)
- **Others:** No permissions at all (`---`)

**Why?** Your home directory is private. Your group can browse it, but strangers can't.

---

## Part 3: Processes & Signals

### Mental Model
A **process** is like a person in an office:
- **PID** = Employee ID (unique)
- **Parent** = Your manager (who started you)
- **Owner** = The person who signed the paycheck
- **Signals** = Messages you can receive

### Viewing All Processes

**Command:** `ps aux`
```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.5  24468 15612 ?        Ss   13:35   0:01 /sbin/init
root          59  0.0  0.5  50452 16812 ?        S<s  13:35   0:00 /usr/lib/systemd/systemd-journald
umer_mu+     460  0.0  0.1   6212  5476 pts/0    Ss   13:35   0:00 /bin/bash
```

### Reading `ps aux`

| Column | Meaning |
|--------|---------|
| `USER` | Who owns this process |
| `PID` | Process ID (unique number) |
| `%CPU` | % of CPU being used RIGHT NOW |
| `%MEM` | % of RAM being used RIGHT NOW |
| `VSZ` | Virtual memory allocated |
| `RSS` | Actual RAM in use |
| `TTY` | Terminal (`?` = no terminal, background service) |
| `STAT` | Status (S=sleeping, R=running) |
| `COMMAND` | What command started it |

### Key Insight: PID 1

```
root           1  0.0  0.5  24468 15612 ?        Ss   13:35   0:01 /sbin/init
```

**PID 1 is special.** It's the parent of ALL processes. Everything started from here.

### Viewing Process Family Tree

**Command:** `pstree -p`
```
systemd(1)─┬─agetty(225)
           ├─chronyd-starter(157)───chronyd(236)───chronyd(243)
           ├─cron(158)
           ├─dbus-daemon(159)
           ├─init-systemd(Ub(2)─┬─SessionLeader(455)───Relay(460)(456)───bash(460)───pstree(1051)
```

**What this shows:** The parent-child relationships. Your bash (460) is a child of SessionLeader (455).

---

## Part 4: Signals — Controlling Processes

### The Two Main Signals

| Signal | Name | Effect | When to use |
|--------|------|--------|-----------|
| SIGTERM (default) | Terminate gracefully | Process can clean up before dying | Normal shutdown |
| SIGKILL (-9) | Kill forcefully | Process dies immediately, no cleanup | Process is hung |

### Practical Example: Starting and Killing

**Step 1: Start a long-running process in background**

```bash
sleep 1000 &
```

Output:
```
[2] 1075
```

The `&` means "run in background". You get a PID (1075).

---

**Step 2: Find the process**

```bash
ps aux | grep sleep
```

Output:
```
umer_mu+    1060  0.0  0.2  16112  7580 pts/0    S    14:14   0:00 sleep 1000
umer_mu+    1080  0.0  0.0   4128  2352 pts/0    S+   14:18   0:00 grep --color=auto sleep
```

---

**Step 3: Kill gracefully (SIGTERM)**

```bash
kill 1075
```

Output after command completes:
```
[2]+  Terminated                 sleep 1000
```

The process shut down gracefully. **SIGTERM worked.**

---

**Step 4: If that doesn't work, kill forcefully (SIGKILL)**

```bash
kill -9 1060
```

Output after command completes:
```
[1]+  Killed                     sleep 1000
```

**Notice the difference:**
- SIGTERM says **"Terminated"** (allowed cleanup)
- SIGKILL says **"Killed"** (forced, no cleanup)

### Why This Matters for Backend Work

- A **Node.js server** receiving SIGTERM closes database connections and flushes logs (graceful)
- A **hung server** getting SIGTERM doesn't respond, so you send SIGKILL (emergency)
- A **containerized app** with PID 1 is special — its signal handling is critical

---

## Part 5: Interactive Process Monitoring

### The `htop` Tool

**Command:** `htop`

This opens an interactive process viewer showing:
- CPU bars and RAM usage at top
- Real-time list of processes sorted by CPU/MEM usage
- Ability to kill processes with keyboard shortcuts (`k`)

**Why it's better than `ps aux`:**
- Real-time updates
- Sorted by what matters (CPU/MEM)
- Interactive (can kill processes mid-view)
- Colorful (easier to read)

**Key columns you care about:**
- **PID** — Process ID
- **USER** — Owner
- **%CPU** — CPU usage (high = problem)
- **%MEM** — Memory usage (high = problem)
- **COMMAND** — What is it?

**To exit:** Press `q`

---

## Part 6: The Unix Superpower — Pipes and Text Processing

### Mental Model: Three Streams

Every process has three communication channels:

| Stream | What it is | Default |
|--------|-----------|---------|
| **stdin** | Input (data going IN) | Your keyboard |
| **stdout** | Output (data coming OUT) | Your screen |
| **stderr** | Errors | Your screen |

### Basic Redirection: `>`

**Command:** Redirect stdout to a file
```bash
echo "Hello World" > /tmp/test.txt
```

Verify it worked:
```bash
cat /tmp/test.txt
```

Output:
```
Hello World
```

**What happened:** The output didn't go to screen, it went to the file.

---

### The Pipe: `|`

The pipe **chains commands together** — stdout of left → stdin of right.

```
Process 1 (output) ──pipe──> Process 2 (input)
```

**Example: Create a file with fruits**

```bash
echo "apple
banana
cherry" > /tmp/fruits.txt
```

**Count the lines:**
```bash
cat /tmp/fruits.txt | wc -l
```

Output:
```
3
```

**What happened:**
1. `cat` reads the file (3 lines)
2. `|` sends output to `wc`
3. `wc -l` counts lines → outputs `3`

---

**Sort the fruits:**
```bash
cat /tmp/fruits.txt | sort
```

Output:
```
apple
banana
cherry
```

---

### Chaining Multiple Pipes (The Real Power)

**Create a log file:**
```bash
echo "500 error
200 ok
500 error
404 not found
200 ok
500 error" > /tmp/log.txt
```

**Solve: "How many 500 errors are in this log?"**

```bash
cat /tmp/log.txt | grep "500" | wc -l
```

Output:
```
3
```

**Breaking it down:**
1. `cat /tmp/log.txt` → outputs all 6 lines
2. `| grep "500"` → filters to only lines with "500" (3 lines remain)
3. `| wc -l` → counts those lines → `3`

---

### Text Processing Tools

#### `grep` — Filter by pattern

```bash
cat /tmp/log.txt | grep "500"
```

Output:
```
500 error
500 error
500 error
```

**Use case:** Find all errors, find all requests to a specific endpoint, etc.

---

#### `wc` — Count lines, words, characters

```bash
cat /tmp/log.txt | wc -l
```

Output:
```
6
```

(6 lines total)

---

#### `sort` — Alphabetize

```bash
cat /tmp/fruits.txt | sort
```

Output:
```
apple
banana
cherry
```

---

#### `uniq -c` — Count unique occurrences

**First, create a file with status codes:**
```bash
echo "200
201
200
404
500" > /tmp/statuses.txt
```

**Count each status:**
```bash
cat /tmp/statuses.txt | sort | uniq -c
```

Output:
```
      2 200
      1 201
      1 404
      1 500
```

**What happened:**
1. `sort` arranges them (so same codes are together)
2. `uniq -c` counts identical lines
3. Output shows: 2 × 200, 1 × 201, etc.

---

#### `cut` — Extract specific columns

Create a realistic log:
```bash
echo "2024-08-22 10:15:23 GET /api/users 200
2024-08-22 10:15:45 POST /api/users 201
2024-08-22 10:16:02 GET /api/users 200
2024-08-22 10:16:18 DELETE /api/orders 404
2024-08-22 10:16:35 GET /api/users 500" > /tmp/access.log
```

**Extract the HTTP method (column 3):**
```bash
cat /tmp/access.log | cut -d ' ' -f 3
```

Output:
```
GET
POST
GET
DELETE
GET
```

**What happened:**
- `-d ' '` = use space as delimiter
- `-f 3` = extract field 3 (3rd column)

---

**Extract the status code (column 5):**
```bash
cat /tmp/access.log | cut -d ' ' -f 5
```

Output:
```
200
201
200
404
500
```

---

**Count occurrences of each status:**
```bash
cat /tmp/access.log | cut -d ' ' -f 5 | sort | uniq -c
```

Output:
```
      2 200
      1 201
      1 404
      1 500
```

**In one line, you just answered:** "What HTTP status codes appeared in my log, and how many of each?"

---

#### `awk` — Powerful text processing

**Extract status >= 500 with timestamp:**
```bash
cat /tmp/access.log | awk '$NF >= 500 {print $1, $NF}'
```

Output:
```
2024-08-22 500
```

**What happened:**
- `$NF` = last field (status code)
- `>= 500` = if status is 500 or higher
- `print $1, $NF` = print 1st field (timestamp) and last field (status)

---

## Part 7: Capstone Challenge

### The Problem

**Count how many GET requests returned 200 status in the access log.**

### The Solution

```bash
cat /tmp/access.log | grep "GET" | grep "200" | wc -l
```

Output:
```
2
```

### Breaking It Down

1. **`cat /tmp/access.log`** → Outputs all 5 log lines
2. **`| grep "GET"`** → Filters to only GET requests (3 remain)
3. **`| grep "200"`** → Filters to only those with status 200 (2 remain)
4. **`| wc -l`** → Counts them → `2`

**This is a real production skill.** In 3 months, you'll do this with 1M line logs on live servers.

---

## Key Takeaways

✅ **Linux has ONE filesystem tree** starting at `/`

✅ **Every file has owner, group, and permissions** — understand `rwxr-xr-x`

✅ **Every process has a PID and parent** — trace relationships with `pstree`

✅ **Signals control processes** — SIGTERM (nice) vs SIGKILL (force)

✅ **Pipes connect tools together** — the Unix philosophy

✅ **Chain small tools** (`grep`, `cut`, `sort`, `uniq`, `wc`, `awk`) to solve big problems **in one line**

✅ **You just solved real production problems** — you're thinking like an engineer now

---

## What's Next: Week 1, Day 2

Tomorrow we'll learn:
- **Shell navigation & file management** (`cd`, `mkdir`, `cp`, `mv`, `rm`, `find`)
- **Text editing** (`vim`/`nano`) — the tools senior engineers use
- **File permissions in depth** (`chmod`, `chown`) — when things break
- **Putting it all together** — your first real project

---

## Practice Commands Reference

Here are all commands from today to reference and practice:

```bash
# Filesystem exploration
ls /
ls /home
ls /mnt
ls /bin | head -20

# File permissions
ls -la ~

# Processes
ps aux
ps aux | grep bash
pstree -p

# Signals & background jobs
sleep 1000 &
ps aux | grep sleep
kill 1075
kill -9 1060
htop

# Redirection
echo "Hello World" > /tmp/test.txt
cat /tmp/test.txt

# Pipes & text processing
echo "apple
banana
cherry" > /tmp/fruits.txt
cat /tmp/fruits.txt | wc -l
cat /tmp/fruits.txt | sort

echo "500 error
200 ok
500 error
404 not found
200 ok
500 error" > /tmp/log.txt
cat /tmp/log.txt | grep "500" | wc -l

echo "2024-08-22 10:15:23 GET /api/users 200
2024-08-22 10:15:45 POST /api/users 201
2024-08-22 10:16:02 GET /api/users 200
2024-08-22 10:16:18 DELETE /api/orders 404
2024-08-22 10:16:35 GET /api/users 500" > /tmp/access.log

cat /tmp/access.log | cut -d ' ' -f 5 | sort | uniq -c
cat /tmp/access.log | awk '$NF >= 500 {print $1, $NF}'
cat /tmp/access.log | grep "GET" | grep "200" | wc -l
```

---

## Reflection Questions

Before Day 2, ask yourself:

1. **Can you explain what a PID is?** (It's a process ID, a unique number)
2. **Can you read file permissions like `rwxr-xr-x`?** (Owner, group, others)
3. **Can you build a pipe with 3+ commands?** (cat | grep | wc)
4. **Do you understand why pipes matter?** (Solve production problems in one line)

If you can answer these, **you're ready for Day 2.** If not, re-read the relevant section.

---

**You've completed Week 1, Day 1 like a pro.** 🚀 Rest well — tomorrow we go deeper.
